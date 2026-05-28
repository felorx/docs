# 通用 AI 视觉与结构化抽取能力设计

## 背景

小汪盘点当前使用 `MockInventoryRecognitionService` 生成固定资产草稿，无法完成真实图片识别。API 项目已有 `AiChat`、`AiImage`、`AiVideo`、`AiJob` 等通用 AI 网关雏形，但 provider 路由仍固定到 `MockAiProvider`，尚未支持真实服务商管理、能力路由和密钥配置。

本设计将图片识别、OCR、视觉标签、商品识别和结构化抽取沉淀为服务端通用能力。小汪盘点只是第一个调用方，不新增面向小汪盘点的专属 `InventoryRecognitionAppService`。

## 目标

- 服务端提供可被任意子应用调用的通用 AI 能力：视觉分析、OCR、图片标签、商品候选、结构化 JSON 抽取。
- 优先支持中国国内服务：腾讯云 OCR、腾讯云图像识别、腾讯混元或其他 OpenAI-compatible 大模型。
- developer 项目提供平台级 AI 服务商管理、模型管理、启用禁用和默认能力切换。
- 小汪盘点把拍照/截图识别流程从 mock 切到真实 API，并保留现有确认、编辑、保存体验。
- AI 密钥只保存在服务端，不暴露给 Flutter 客户端。

## 非目标

- 不新增只服务于小汪盘点的专用后端识别接口。
- 不在 Flutter 端直接调用腾讯云或大模型厂商接口。
- 不要求第一阶段实现完整的多租户/每开发者独立密钥；先做平台全局配置。
- 不把服务商管理做成静态配置文件；developer 需要能管理和切换。

## 参考服务

- 腾讯云 OCR 通用印刷体识别：`GeneralBasicOCR`，适合截图、文档、页面文字识别。官方文档：https://cloud.tencent.com/document/product/866/33526
- 腾讯云图像识别：`DetectLabelPro` 用于通用图像标签，`DetectProduct` 用于商品识别。API 概览：https://cloud.tencent.cn/document/product/865/35462
- 腾讯混元 OpenAI 兼容接口：用于复用 OpenAI-compatible chat 调用路径。官方文档：https://cloud.tencent.com/document/product/1729/111007
- 后续可扩展通义千问、DeepSeek、火山方舟、OpenAI-compatible 聚合服务等。

## 总体架构

### 服务端能力层

API 新增通用 AI 能力模块，保持与现有 `Puupees.Ai` 命名空间一致：

- `AiProviderAppService`：管理平台 AI 服务商和模型配置。
- `AiVisionAppService`：统一图片分析能力。
- `AiExtractionAppService`：统一结构化抽取能力。
- `IAiProviderRouter`：从固定 mock 路由升级为按能力、模型、默认配置路由。
- Provider adapter：
  - `TencentCloudVisionProvider`：OCR、图像标签、商品识别。
  - `OpenAiCompatibleChatProvider`：通用 chat / 结构化抽取。
  - `MockAiProvider`：测试和未配置时兜底。

### 客户端调用层

小汪盘点新增 `ApiInventoryRecognitionService`，实现现有 `InventoryRecognitionService` 接口：

1. 拍照或选择截图。
2. 上传图片到对象存储，得到 `AiMediaReferenceDto` 可引用的 URL 或 `bucket + key`。
3. 调用 `AiVisionAppService.AnalyzeImageAsync`。
4. 调用 `AiExtractionAppService.CreateStructuredExtractionAsync`，传入小汪盘点资产草稿 JSON Schema。
5. 将返回 JSON 映射为 `InventoryAssetDraft`。
6. 进入现有确认页，用户可编辑、选择、保存。

## 数据模型

### AiProvider

平台服务商实体，建议放在 `Puupees.Domain/Ai`：

- `Id`
- `Name`
- `ProviderType`：`TencentCloud`、`OpenAiCompatible`、`Mock`
- `DisplayName`
- `BaseUrl`
- `Region`
- `Enabled`
- `Capabilities`：字符串集合，包含 `chat`、`vision`、`ocr`、`image-label`、`product-detect`、`structured-extraction`
- `SecretConfigured`：DTO 只返回是否已配置
- `EncryptedSecret`：服务端加密保存，DTO 不返回明文
- `Metadata`：JSON 字典，存放 provider 特有配置

### AiModel

模型配置第一阶段就使用独立实体，便于 developer 管理、能力默认切换和后续用量统计：

- `Id`
- `ProviderId`
- `Name`
- `DisplayName`
- `Capabilities`
- `Enabled`
- `IsDefault`
- `DefaultParameters`：JSON 字典，例如 temperature、maxTokens

默认模型约束：同一能力在启用模型中最多一个默认项。

## API 设计

### AiProviderAppService

- `CreateAsync(CreateOrUpdateAiProviderDto input)`
- `UpdateAsync(Guid id, CreateOrUpdateAiProviderDto input)`
- `DeleteAsync(Guid id)`
- `GetAsync(Guid id)`
- `GetListAsync(AiProviderGetListInput input)`
- `SetEnabledAsync(Guid id, bool enabled)`
- `SetDefaultModelAsync(Guid modelId, string capability)`
- `TestAsync(Guid id, TestAiProviderDto input)`

密钥更新规则：

- 创建时可传密钥。
- 更新时如果密钥字段为空且 `clearSecret` 为 false，则保留旧密钥。
- DTO 只返回 `secretConfigured`，不返回密钥明文。

### AiVisionAppService

`AnalyzeImageAsync(CreateAiImageAnalysisDto input)` 输入：

- `Image`：`AiMediaReferenceDto`
- `Tasks`：`ocr`、`labels`、`products`、`caption`
- `Locale`：默认 `zh-CN`
- `Provider`：可选；为空时按能力路由默认 provider
- `Model`：可选
- `Metadata`

输出 `AiImageAnalysisDto`：

- `OcrTexts`：文本、位置、置信度
- `Labels`：标签名、分类、置信度
- `Products`：商品候选名、品牌、分类、置信度
- `Caption`：图片概述，可由 vision-capable 大模型或标签摘要生成
- `ProviderResults`：各 provider 的名称、模型、原始 request id、耗时
- `Usage`

### AiExtractionAppService

`CreateStructuredExtractionAsync(CreateAiStructuredExtractionDto input)` 输入：

- `SchemaName`
- `JsonSchema`
- `SourceText`
- `SourceVisionResult`
- `Instruction`
- `Provider`
- `Model`
- `Metadata`

输出 `AiStructuredExtractionDto`：

- `Json`
- `Confidence`
- `MissingFields`
- `RawText`
- `Provider`
- `Model`
- `Usage`

结构化抽取必须要求模型只返回 JSON。服务端解析失败时要返回明确业务错误，而不是把不可解析文本透给客户端当成功结果。

## Provider 路由

`DefaultAiProviderRouter` 改为：

1. 如果输入指定 provider/model，优先使用指定项，并校验启用和能力匹配。
2. 如果只指定 model，根据模型表查找。
3. 如果未指定，按 capability 查找默认启用模型。
4. 未配置真实 provider 时返回 mock provider，测试环境保持稳定。
5. provider 调用失败时只在配置允许 fallback 的能力上回退；默认不静默降级，避免用户误以为是真实识别结果。

能力匹配示例：

- `ocr` -> 腾讯云 OCR provider。
- `image-label`、`product-detect` -> 腾讯云图像识别 provider。
- `structured-extraction` -> OpenAI-compatible chat provider。
- `caption` -> vision-capable chat provider；没有视觉模型时由标签和 OCR 摘要生成。

## 小汪盘点集成

### 物理资产照片

任务列表：`products`、`labels`、`ocr`、`caption`

抽取 schema 包含：

- `name`
- `description`
- `assetKind`
- `status`
- `source`
- `category`
- `brand`
- `model`
- `serialNumber`
- `location`
- `currency`
- `purchasePrice`
- `currentValue`
- `purchaseDate`
- `warrantyEndDate`
- `importance`
- `reminderEnabled`
- `tags`
- `fieldConfidence`
- `missingFields`

### 虚拟资产截图

任务列表：`ocr`、`labels`、`caption`

抽取重点：

- 服务名、计划名、账号提示、价格、货币、续费日期、计费周期。
- 如果 OCR 能识别到邮箱、手机号、订单号等敏感信息，抽取时只生成 `accountHint`，避免保存完整隐私文本。

### 失败体验

- 图片上传失败：停留在选择/拍照页，显示“上传失败，请重试”。
- 视觉分析失败：显示“图片识别失败”，允许重试或换图。
- 结构化抽取失败：显示“生成资产草稿失败”，保留识别文本供下一次重试不需要重新上传。
- 返回空草稿：显示“未识别到资产”，提供重新拍照/选择截图。

## developer 管理端

新增路由：`/developer/development/ai-providers`

入口放在“开发”页，与 API Key / SDK 同级。

### 列表页

展示：

- 名称
- 类型
- 启用状态
- 默认能力标签
- 模型数量
- 更新时间

操作：

- 新增
- 编辑
- 启用/禁用
- 删除
- 测试连接

### 编辑页

字段：

- 名称
- 类型
- Base URL
- Region
- 能力多选
- SecretId / SecretKey 或 ApiKey
- 模型列表：模型名、能力、是否默认、默认参数
- 备注

密钥保存后不回显明文，只显示“已配置”。

## 安全

- API provider 管理接口必须 `[Authorize]`。
- 后续可补 `PuupeePermissions.AiProvider.*`，包括查看、创建、更新、删除、测试。
- 密钥使用 ABP 设置加密能力或独立加密服务保存；DTO 禁止返回明文。
- 调用 provider 时不要在日志输出密钥、完整图片 URL、原始 OCR 敏感文本。
- structured extraction 的 raw response 可以保存到返回 DTO，但不落库，避免长期保存用户隐私。

## 测试策略

### API 单元测试

- provider 创建、更新密钥不回显、保留旧密钥、清空密钥。
- router 按指定 model、默认能力、未配置 fallback 选择正确 provider。
- `AiVisionAppService` 在 mock provider 下返回 OCR/标签/商品结构。
- `AiExtractionAppService` 能解析 JSON，解析失败时返回业务错误。

### API 集成测试

- controller 路由和授权检查。
- OpenAI-compatible 请求映射到现有 chat provider。
- 腾讯云 adapter 使用 fake HTTP handler 验证签名请求、响应映射和错误映射。

### Flutter 测试

- `ApiInventoryRecognitionService` 能把 vision + extraction DTO 映射为 `InventoryAssetDraft`。
- 现有确认页测试保留，并替换 provider override 验证 API 服务调用路径。
- 上传失败、识别失败、空草稿三类 UI 状态。

### developer 测试

- 列表页展示 provider。
- 编辑页保存时不要求重新输入已配置密钥。
- 启用/禁用和默认能力切换会刷新列表。

## 迁移与上线

1. 保留 `MockAiProvider`，保证无配置环境测试通过。
2. 新增数据库表和迁移：`AiProviders`、`AiModels`。
3. developer 先支持基本 CRUD、启用状态、默认能力切换。
4. API 先接腾讯云 OCR / 图像识别和 OpenAI-compatible chat。
5. 小汪盘点从 mock 切到 `ApiInventoryRecognitionService`，并保留测试 override。
6. 后续再补更细权限、调用审计、用量统计、成本控制。

## 验收标准

- developer 可创建腾讯云和 OpenAI-compatible provider，并切换默认能力。
- 配置腾讯云密钥后，`AnalyzeImageAsync` 可返回真实 OCR/标签/商品候选。
- 配置大模型 provider 后，`CreateStructuredExtractionAsync` 可返回合法 JSON。
- 小汪盘点拍照或选择截图后能通过真实 API 生成资产草稿，并进入原确认页保存。
- 未配置真实 provider 时，测试环境仍可通过 mock 行为稳定运行。
