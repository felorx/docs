# Felorx Sentry MVP 设计

## 背景

`git@github.com:felorx/felorx-sentry.git` 当前是独立的 Go API + Next.js 后台项目，目标是实现兼容 Sentry SDK 的事件采集服务和可视化后台。现有实现可以接收 envelope 并按 `project_id` 查看事件，但缺少真正的项目管理、Issue 聚合、日志、Trace 链路、权限和后台信息架构。

本设计定义第一期可用 MVP：把 `felorx-sentry` 作为主仓库的独立 submodule 接入，并在该仓库内完成模块化重构和后台功能补齐。MVP 目标是能闭环「项目管理 / 应用管理、Issues、日志、整体链路」。

## 已确认决策

- 形态：独立 Sentry 兼容服务，不并入 Felorx 主 .NET API。
- 主仓库路径：添加为 `apps/sentry` submodule。
- 认证：后台接入 Felorx Auth/OIDC 登录。
- 项目模型：`felorx-sentry` 维护独立 Project，可选绑定 Felorx App。
- 范围：第一期做可用 MVP，不做告警、Source Map、Symbolication、团队组织、通知和 ClickHouse/OpenSearch。
- 架构：保留 Go + Gin + PostgreSQL + Next.js 技术栈，先模块化重构，不引入队列。
- 协议：和 Sentry 官方数据结构保持一致。内部可以有索引列和聚合表，但原始 payload、envelope item 和官方字段必须完整保留。

## 目标

- 支持创建、编辑、禁用和查看监控项目。
- 支持为项目生成 DSN，并兼容常见 Sentry SDK 上报路径。
- 支持接收 Sentry envelope、store event、log item、transaction payload 和 span v2 item。
- 支持按官方 event payload 字段保存和查看单次事件。
- 支持自动按 fingerprint 聚合 Issue，并维护 unresolved、resolved、ignored 状态。
- 支持日志列表和日志详情，日志可通过 `trace_id`、`span_id` 和事件关联。
- 支持 trace 查询，展示同一 trace 下的 transaction、span、log 和 error event。
- 支持 Felorx 登录后的项目权限校验。
- 用测试覆盖协议解析、字段抽取、Issue 聚合和关键 API。

## 非目标

- 不复刻完整 sentry.io 产品。
- 不实现告警规则、通知渠道、Release Health、Replay、Profiling、Source Map、debug symbol、symbolication。
- 不引入 ClickHouse、OpenSearch、Kafka、Redis queue 或异步 worker。
- 不把 Felorx 主 .NET API 变成事件存储后端。
- 不在 MVP 中实现组织、团队、审计日志和复杂 RBAC。
- 不丢弃未知 Sentry envelope item。

## 仓库接入

主仓库添加 submodule：

```bash
git submodule add -b main git@github.com:felorx/felorx-sentry.git apps/sentry
```

`.gitmodules` 使用现有风格：

```ini
[submodule "apps/sentry"]
	path = apps/sentry
	url = git@github.com:felorx/felorx-sentry.git
	branch = main
```

`felorx-sentry` 内部继续独立维护 Go API、Next.js web、Dockerfile 和 docker-compose。主仓库只保存 submodule 指针和必要的构建入口配置。

## 总体架构

`felorx-sentry` 拆成以下模块：

- `auth`：Felorx OIDC 登录、session/JWT 校验、当前用户解析。
- `projects`：Project CRUD、DSN、可选 Felorx App 绑定、成员角色。
- `ingest`：Sentry 兼容上报端点、DSN/project 校验、envelope/store 解析。
- `events`：官方 event payload 保存、索引字段抽取、事件查询。
- `issues`：fingerprint 计算、Issue upsert、状态流转。
- `logs`：官方 `log` envelope item 和兼容日志事件查询。
- `traces`：transaction、span v2、trace 关联查询。
- `shared`：分页、错误响应、时间解析、JSONB 工具、测试 fixture。

Go API 采用 handler -> service -> repository 分层。Handler 只负责 HTTP 参数和响应；Service 承载业务规则；Repository 封装 GORM/PostgreSQL 查询。

第一期 ingest 同步执行：

1. 解析 DSN/project。
2. 解析 envelope 或 store payload。
3. 保存 envelope item 原始数据。
4. 按 item type 分发到 event/log/transaction/span 处理器。
5. 抽取索引字段。
6. upsert issue/trace/log。
7. 返回 Sentry SDK 可接受的响应。

## Sentry 官方协议兼容

实现以官方协议为准：

- [Sentry Envelopes](https://develop.sentry.dev/sdk/foundations/envelopes/)
- [Event Payloads](https://develop.sentry.dev/sdk/foundations/envelopes/event-payloads/)
- [Transaction Payloads](https://develop.sentry.dev/sdk/foundations/envelopes/event-payloads/transaction/)
- [Span Protocol](https://develop.sentry.dev/sdk/telemetry/spans/span-protocol/)
- [Logs](https://develop.sentry.dev/sdk/telemetry/logs/)

兼容原则：

- Envelope header 和 item header 必须作为 JSONB 保存。
- Item payload 必须原样保存；未知 item 也必须保存。
- 支持 `length` item payload，不能再按简单换行拆分所有 item。
- 支持整个请求由 `Content-Encoding: gzip` 压缩。
- Store endpoint 是 deprecated，但 MVP 继续兼容常见旧 SDK。
- API 响应可以做 Felorx 后台 DTO，但事件详情必须能返回官方原始 payload。

### Event Payload

`events.raw_payload` 保存完整 event JSONB。索引列只从官方字段抽取：

- `event_id`
- `timestamp`
- `platform`
- `level`
- `logger`
- `transaction`
- `server_name`
- `release`
- `dist`
- `environment`
- `fingerprint`
- `message`
- `exception_type`
- `trace_id`
- `span_id`
- `user_id`

详情页按官方接口展示：

- `exception`
- `stacktrace`
- `threads`
- `request`
- `user`
- `tags`
- `contexts`
- `breadcrumbs`
- `debug_meta`
- `sdk`
- `modules`
- `extra`
- `errors`

### Log Payload

优先支持官方 `log` envelope item：

```json
{
  "type": "log",
  "item_count": 5,
  "content_type": "application/vnd.sentry.items.log+json"
}
```

payload 为：

```json
{
  "version": 2,
  "ingest_settings": {
    "infer_ip": "auto",
    "infer_user_agent": "auto"
  },
  "items": [
    {
      "timestamp": 1746456149.0191,
      "trace_id": "624f66e93a04469f9992c7e9f1485056",
      "span_id": "b0e6f15b45c36b12",
      "level": "info",
      "body": "User John has logged in!",
      "severity_number": 9,
      "attributes": {}
    }
  ]
}
```

索引列只抽取 `timestamp`、`trace_id`、`span_id`、`level`、`body`、`severity_number`。`attributes` 和完整 log item 原样保存为 JSONB。

兼容策略：没有 `log` item 的旧事件，如果 `type=event` 且主要是 message/logger，也可以在事件详情中显示为日志来源，但不伪造官方 log item。

### Transaction 与 Span

兼容两类来源：

- 旧 transaction payload：`type=transaction`，使用 `contexts.trace` 和 `spans[]`。
- span v2 envelope item：`type=span`，`content_type=application/vnd.sentry.items.span.v2+json`，payload `items[]`。

索引字段：

- `trace_id`
- `span_id`
- `parent_span_id`
- `name` 或 `transaction`
- `op` 或 `attributes.sentry.op`
- `status`
- `is_segment`
- `start_timestamp`
- `end_timestamp`
- `duration_ms`
- `release`
- `environment`

完整 transaction payload、span item、attributes、links 和 measurements 均保留 JSONB。

## 数据模型

### `projects`

- `id` uuid primary key
- `slug` text unique
- `name` text
- `platform` text nullable
- `status` text (`active`, `disabled`)
- `dsn_public_key` text unique
- `dsn_secret_key_hash` text nullable
- `felorx_app_id` text nullable
- `felorx_app_name` text nullable
- `created_by_felorx_user_id` text
- `created_at`, `updated_at`

### `project_members`

- `id` uuid primary key
- `project_id` uuid
- `felorx_user_id` text
- `email` text nullable
- `role` text (`owner`, `admin`, `member`, `viewer`)
- `created_at`, `updated_at`

### `envelopes`

- `id` uuid primary key
- `project_id` uuid
- `event_id` text nullable
- `dsn` text nullable
- `sdk` jsonb nullable
- `sent_at` timestamptz nullable
- `headers` jsonb not null
- `received_at` timestamptz not null
- `remote_addr` text nullable
- `user_agent` text nullable

### `envelope_items`

- `id` uuid primary key
- `envelope_id` uuid nullable
- `project_id` uuid
- `item_type` text
- `content_type` text nullable
- `item_count` int nullable
- `headers` jsonb not null
- `payload` jsonb nullable
- `payload_text` text nullable
- `payload_bytes` bytea nullable
- `payload_size` bigint
- `created_at`

### `events`

- `id` uuid primary key
- `project_id` uuid
- `issue_id` uuid nullable
- `envelope_item_id` uuid nullable
- `event_id` text unique per project
- `type` text (`event`, `transaction`, etc.)
- `timestamp` timestamptz
- `platform` text nullable
- `level` text
- `logger` text nullable
- `message` text nullable
- `transaction` text nullable
- `server_name` text nullable
- `release` text nullable
- `dist` text nullable
- `environment` text nullable
- `fingerprint` text nullable
- `exception_type` text nullable
- `trace_id` text nullable
- `span_id` text nullable
- `user_id` text nullable
- `raw_payload` jsonb not null
- `created_at`

### `issues`

- `id` uuid primary key
- `project_id` uuid
- `fingerprint` text
- `title` text
- `culprit` text nullable
- `exception_type` text nullable
- `level` text
- `status` text (`unresolved`, `resolved`, `ignored`)
- `event_count` bigint
- `first_seen` timestamptz
- `last_seen` timestamptz
- `last_event_id` uuid nullable
- `metadata` jsonb nullable
- unique `(project_id, fingerprint)`

### `logs`

- `id` uuid primary key
- `project_id` uuid
- `envelope_item_id` uuid nullable
- `event_id` uuid nullable
- `timestamp` timestamptz
- `trace_id` text nullable
- `span_id` text nullable
- `level` text
- `body` text
- `severity_number` int nullable
- `attributes` jsonb nullable
- `raw_payload` jsonb not null
- `created_at`

### `traces`

- `id` uuid primary key
- `project_id` uuid
- `trace_id` text
- `name` text nullable
- `operation` text nullable
- `status` text nullable
- `start_timestamp` timestamptz nullable
- `end_timestamp` timestamptz nullable
- `duration_ms` double precision nullable
- `root_event_id` uuid nullable
- `created_at`, `updated_at`
- unique `(project_id, trace_id)`

### `spans`

- `id` uuid primary key
- `project_id` uuid
- `trace_id` text
- `span_id` text
- `parent_span_id` text nullable
- `name` text
- `operation` text nullable
- `status` text nullable
- `is_segment` bool
- `start_timestamp` timestamptz
- `end_timestamp` timestamptz
- `duration_ms` double precision
- `attributes` jsonb nullable
- `links` jsonb nullable
- `raw_payload` jsonb not null
- unique `(project_id, trace_id, span_id)`

## Issue 聚合

Fingerprint 规则：

1. 如果 payload 有官方 `fingerprint`，使用其 JSON canonical string 后 hash。
2. 如果是异常事件，使用 `exception.values[-1].type`、`exception.values[-1].value`、top in-app frame 的 `filename/function/lineno` 生成 hash。
3. 如果有 stacktrace 但无 exception，使用 top in-app frame + message。
4. 否则使用 `logger + message` 或 `transaction + message`。

Issue 标题优先级：

1. `exception.values[-1].type + ": " + exception.values[-1].value`
2. `message`
3. `transaction`
4. `event_id`

事件写入后同步 upsert `issues`：

- 新 fingerprint 创建 unresolved issue。
- 已存在 issue 增加 `event_count`，更新 `last_seen`、`last_event_id`、`level`。
- 如果 issue 为 resolved，MVP 中再次出现时自动回归 unresolved。
- ignored issue 再出现仍保持 ignored，但更新计数和 last_seen。

## API 设计

### Ingest

- `POST /api/:project_id/envelope/`
- `POST /api/:project_id/api/store/envelope/`
- `POST /api/:project_id/store/`
- `POST /api/:project_id/api/store/`

`project_id` 可以是项目 slug，也可以是 DSN path project id。服务端根据 path、DSN public key 和项目状态校验项目。

### 后台 API

- `GET /api/projects`
- `POST /api/projects`
- `GET /api/projects/:project_id`
- `PATCH /api/projects/:project_id`
- `POST /api/projects/:project_id/rotate-dsn`
- `GET /api/projects/:project_id/overview`
- `GET /api/projects/:project_id/issues`
- `GET /api/projects/:project_id/issues/:issue_id`
- `PATCH /api/projects/:project_id/issues/:issue_id`
- `GET /api/projects/:project_id/events`
- `GET /api/projects/:project_id/events/:event_id`
- `GET /api/projects/:project_id/logs`
- `GET /api/projects/:project_id/traces`
- `GET /api/projects/:project_id/traces/:trace_id`

列表 API 统一支持：

- `cursor` 或 `page/page_size`
- `query`
- `environment`
- `release`
- `level`
- `start`
- `end`

## 后台页面

Next.js 后台第一期页面：

- 登录页：跳转 Felorx OIDC。
- 项目列表：项目、状态、绑定 App、最近事件数、未解决 Issue 数。
- 项目创建/编辑：slug/name/platform/Felorx App 绑定/状态。
- 项目设置：DSN 展示和轮换。
- 项目概览：错误数、未解决 Issue、日志数、trace 数、最近事件。
- Issues 列表：按 status、level、environment、release、query 过滤。
- Issue 详情：标题、状态、计数、首次/最后出现、最近事件、stacktrace、tags、contexts、breadcrumbs、raw payload。
- 事件详情：官方 event payload 分区展示。
- 日志列表：level、body、trace_id、span_id、timestamp、attributes。
- Trace 列表：trace_id、name、operation、duration、status、关联错误数。
- Trace 详情：transaction/span 树、关联 logs、关联 events。

界面风格应偏后台工具：密度高、可扫描、少装饰，避免营销页式卡片堆叠。

## 认证与权限

后台使用 Felorx OIDC：

- Go API 配置 `FELORX_OIDC_ISSUER`、`FELORX_OIDC_CLIENT_ID`、`FELORX_OIDC_CLIENT_SECRET`、`FELORX_OIDC_REDIRECT_URI`。
- Next.js 通过后台 API 或 NextAuth/Auth.js 完成 OIDC 登录。
- API 从 session/JWT 中解析 `felorx_user_id`、email、name。
- Project 创建者默认 owner。
- MVP 权限：
  - owner/admin：项目管理、DSN 轮换、Issue 状态修改。
  - member：查看、Issue 状态修改。
  - viewer：只读。

Ingest endpoint 不要求用户登录，只用 DSN/project 校验。

## 测试策略

Go API：

- Envelope parser：
  - 支持 `length` payload。
  - 支持无 `length` 的简单 payload。
  - 支持 gzip request body。
  - 未知 item 保留。
- Event extractor：
  - 从官方 event payload 抽取 `event_id/timestamp/platform/level/release/environment/trace_id`。
  - 保留 raw payload。
- Issue grouper：
  - 官方 fingerprint 优先。
  - exception + stack frame 生成稳定 hash。
  - resolved issue 再出现变 unresolved。
  - ignored issue 再出现保持 ignored。
- Log parser：
  - 解析 `log` item 的 `items[]`。
  - 校验 `item_count` 与实际数量。
  - 保存 attributes JSONB。
- Trace parser：
  - 解析 transaction `contexts.trace` 和 `spans[]`。
  - 解析 span v2 `items[]`。
- HTTP API：
  - project CRUD。
  - ingest 后能查到 event/issue/log/trace。
  - 无权限用户不能访问项目后台 API。

Next.js：

- API client 单元测试。
- 页面组件基础渲染测试。
- 关键筛选参数生成测试。
- 构建和类型检查。

验证命令：

```bash
cd apps/sentry && go test ./...
cd apps/sentry/web && npm test
cd apps/sentry/web && npm run lint
cd apps/sentry/web && npm run build
```

## 分阶段实施

### 阶段 1：接入与基线

- 添加 `apps/sentry` submodule。
- 更新 README 和本地启动说明。
- 在 `felorx-sentry` 中增加基础测试框架。
- 修正 Go module path 为 `github.com/felorx/felorx-sentry`。

### 阶段 2：协议解析与数据库

- 重写 envelope parser，支持官方 length payload。
- 建立 PostgreSQL schema 和 migration。
- 实现 event/log/transaction/span extractor。
- 保存 envelope 和 envelope item 原始数据。

### 阶段 3：项目与认证

- 实现 Felorx OIDC 登录。
- 实现 Project CRUD、DSN 生成、项目成员权限。
- 实现后台 API 权限中间件。

### 阶段 4：Issue、日志和 Trace 查询

- 实现 Issue 聚合和状态流转。
- 实现 event/issue/log/trace 列表和详情 API。
- 增加筛选、分页和索引。

### 阶段 5：Next.js 后台重构

- 重做后台信息架构和页面。
- 实现项目、概览、Issues、事件、日志、Trace 页面。
- 接入登录状态和 API client。

### 阶段 6：验证与部署

- 补齐测试 fixture。
- 运行 Go 和 Next.js 验证。
- 更新 Dockerfile/docker-compose。
- 编写部署环境变量说明。

## 风险与缓解

- 风险：Sentry envelope 解析只按换行会破坏带换行 payload。
  缓解：严格按官方 `length` 解析，并补测试。

- 风险：自定义字段设计偏离 Sentry，后续 SDK 不兼容。
  缓解：raw payload 永久保留，索引列只派生官方字段。

- 风险：同步 ingest 在高吞吐下变慢。
  缓解：MVP 保持同步，模块边界预留后续 worker；先用索引和事务优化。

- 风险：Felorx OIDC 接入阻塞本地开发。
  缓解：提供 dev auth 模式，仅本地启用，通过固定 dev user 模拟登录。

- 风险：Issue grouping 和官方 Sentry 不完全一致。
  缓解：MVP 明确优先官方 `fingerprint`，默认 grouping 只做稳定近似，并在代码里隔离 grouper 以便后续替换。

- 风险：日志协议更新较快。
  缓解：保存完整 `log` item，索引字段保持最小，未知 attributes 不做丢弃。

## 验收标准

- 主仓库存在 `apps/sentry` submodule，指向 `git@github.com:felorx/felorx-sentry.git`。
- `felorx-sentry` 可通过 docker-compose 启动 API、web、PostgreSQL。
- 使用标准 Sentry DSN 向 `/api/:project_id/envelope/` 上报 event 后，后台能看到 event 和聚合 issue。
- 上报包含官方 `fingerprint` 的两个 event 会聚合到同一 issue。
- 上报官方 `log` item 后，后台日志列表能按 `level` 和 `trace_id` 查询。
- 上报 transaction 或 span v2 后，后台 trace 详情能展示 span 树，并关联同 trace 的日志和错误事件。
- Issue 可以在 unresolved、resolved、ignored 间切换。
- 事件详情能查看官方 raw payload，且包含 `exception/request/user/tags/contexts/breadcrumbs` 等字段时不丢失。
- 未登录用户不能访问后台 API；无项目权限用户不能读取项目数据。
- `go test ./...`、`npm run lint`、`npm run build` 通过。
