# Puupee AI Gateway 设计规格

日期：2026-05-14

## 背景

小汪手工后续需要真实图片生成、图生图、裁剪图生成和过程视频生成。客户端不能直接调用 OpenAI、Google、Runway、fal、Replicate 或自建模型服务，因为这样会暴露第三方 API Key，也会让供应商切换、计费、限流、审计和失败重试散落在各个客户端里。

因此 API 项目需要新增统一的 AI Gateway。客户端始终调用 Puupee 后端接口，后端负责选择实际供应商、保存密钥、转换请求和返回统一结果。

第一阶段目标是落地稳定的接口契约、OpenAI 兼容外壳、Provider 抽象和 Mock Provider，不接真实第三方服务。这样小汪手工可以先改成调用后端真实接口，后续替换 Provider 时不需要重写 Flutter 业务流程。

## 目标

- 在 `api` 项目新增公共 AI 接口，覆盖 chat、文生图、图生图、文生视频、图生视频。
- 提供 OpenAI 兼容入口，方便后端或内部工具复用 OpenAI 风格请求和 SDK 生态。
- 提供 Puupee 原生入口，方便客户端使用更稳定、更适合业务的统一 DTO。
- 将第三方 API Key、供应商配置和模型映射全部放在后端。
- 媒体生成统一走异步任务模型，返回任务状态和输出媒体引用。
- 第一阶段使用 Mock Provider 返回稳定占位结果，先打通端到端契约。

## 非目标

- 第一阶段不直接接入 OpenAI、Vertex AI、Runway、fal、Replicate 或 ComfyUI。
- 第一阶段不实现真实图片、视频或裁剪算法。
- 第一阶段不实现复杂队列、分布式任务执行器或成本结算系统。
- 第一阶段不实现实时流式视频进度。
- 第一阶段不要求所有 OpenAI 参数 100% 兼容，只兼容常用稳定子集，并允许保留未知扩展字段。

## 总体方案

采用“双接口面”设计：

1. OpenAI 兼容接口面
   - 目标是兼容常见 OpenAI 请求和响应形态。
   - 适合内部工具、调试脚本、兼容 SDK、迁移旧调用。
   - 路由放在显式 MVC Controller 中，避免 ABP conventional controller 生成的路径与 OpenAI 风格不一致。

2. Puupee 原生接口面
   - 目标是给 Flutter 客户端和 Puupee 业务使用。
   - 使用统一媒体任务 DTO，不暴露供应商参数和 API Key。
   - 输入图片优先使用 Puupee 对象存储 key 或 URL 引用，而不是让客户端把第三方上传协议耦合进业务。

后端内部通过 Provider Router 选择实际供应商。第一阶段注册 `MockAiProvider`，后续可以按模型名、租户、应用、用户订阅或配置切换到不同 Provider。

## API 分层

新增目录建议：

```text
api/src/Puupees.Application.Contracts/Ai/
├── IAiChatAppService.cs
├── IAiImageAppService.cs
├── IAiVideoAppService.cs
├── IAiJobAppService.cs
├── Dtos/
│   ├── AiChatDtos.cs
│   ├── AiMediaDtos.cs
│   ├── AiJobDtos.cs
│   └── OpenAiCompatibleDtos.cs

api/src/Puupees.Application/Ai/
├── AiChatAppService.cs
├── AiImageAppService.cs
├── AiVideoAppService.cs
├── AiJobAppService.cs
├── Providers/
│   ├── IAiProviderRouter.cs
│   ├── IAiChatProvider.cs
│   ├── IAiImageProvider.cs
│   ├── IAiVideoProvider.cs
│   └── MockAiProvider.cs

api/src/Puupees.HttpApi/Controllers/Ai/
├── OpenAiCompatibleChatController.cs
├── OpenAiCompatibleImagesController.cs
└── OpenAiCompatibleVideosController.cs
```

`Application.Contracts` 定义客户端可见 DTO 和 AppService 接口。`Application` 定义业务编排、Provider 抽象和 Mock 实现。`HttpApi` 承载固定 `/api/ai/**` 路由、multipart 和未来流式响应的显式 Controller。Controller 只做协议适配和路由稳定性保证，核心逻辑委托给 AppService。

## OpenAI 兼容接口

第一阶段支持以下路由：

```text
POST /api/ai/v1/chat/completions
POST /api/ai/v1/images/generations
POST /api/ai/v1/images/edits
POST /api/ai/v1/videos
GET  /api/ai/v1/videos/{id}
```

### Chat Completions

请求兼容常用字段：

- `model`
- `messages`
- `temperature`
- `top_p`
- `max_tokens`
- `stream`
- `tools`
- `tool_choice`
- `response_format`
- `metadata`

第一阶段 `stream` 可以接收但返回非流式结果。响应保持 OpenAI Chat Completion 形态：

- `id`
- `object = "chat.completion"`
- `created`
- `model`
- `choices`
- `usage`

### Images Generations

请求兼容常用字段：

- `model`
- `prompt`
- `size`
- `quality`
- `n`
- `response_format`
- `metadata`

第一阶段返回 OpenAI Images 风格响应：

- `created`
- `data[]`
  - `url`
  - `b64_json`
  - `revised_prompt`

Mock Provider 默认返回可识别的占位媒体 URL 或 Puupee CDN 占位引用。

### Images Edits

支持两种输入：

- OpenAI 兼容 multipart：`image`、`mask`、`prompt`、`model`。
- Puupee 扩展 JSON：`input_images[]` 使用对象存储 key、bucket、url 或 media id。

第一阶段 Controller 接受 multipart 外壳，但内部统一转换成 `AiImageEditInputDto`。这样客户端推荐走 Puupee 原生 JSON，兼容工具可以走 multipart。

### Videos

视频天然是异步任务。OpenAI 兼容视频入口返回视频任务：

- `id`
- `object = "video.generation"`
- `status`
- `model`
- `created_at`
- `completed_at`
- `error`

`POST /api/ai/v1/videos` 同时支持：

- 文生视频：只有 `prompt`。
- 图生视频：`prompt` 加 `input_images[]` 或 multipart 参考图。

`GET /api/ai/v1/videos/{id}` 查询任务状态和输出视频。

## Puupee 原生接口

第一阶段支持以下固定路由。这些路由由显式 Controller 暴露，再委托给 AppService，避免 ABP conventional controller 生成 `/api/app/**` 风格路径后影响客户端长期兼容。

```text
POST /api/ai/chat/completions
POST /api/ai/images/generations
POST /api/ai/images/edits
POST /api/ai/videos/generations
POST /api/ai/videos/edits
GET  /api/ai/jobs/{id}
POST /api/ai/jobs/{id}/cancel
```

客户端优先调用这些接口。Puupee 原生接口第一阶段只要求 JSON 请求，图片和视频输入使用对象存储引用；multipart 主要保留给 OpenAI 兼容入口和内部调试工具。

### 统一媒体任务 DTO

媒体生成返回统一任务：

```json
{
  "id": "ai_job_01HX...",
  "kind": "image",
  "operation": "image_edit",
  "status": "queued",
  "provider": "mock",
  "model": "mock-image",
  "prompt": "使用上传的蓝色牛仔布制作这个托特包",
  "outputs": [],
  "usage": null,
  "error": null,
  "createdAt": "2026-05-14T00:00:00Z",
  "updatedAt": "2026-05-14T00:00:00Z",
  "completedAt": null
}
```

任务成功后：

```json
{
  "id": "ai_job_01HX...",
  "kind": "video",
  "operation": "image_to_video",
  "status": "succeeded",
  "provider": "mock",
  "model": "mock-video",
  "outputs": [
    {
      "type": "video",
      "url": "https://cdn.puupee.com/ai/mock/process.mp4",
      "storageKey": "ai/mock/process.mp4",
      "mimeType": "video/mp4",
      "width": 1280,
      "height": 720,
      "durationSeconds": 8
    }
  ],
  "completedAt": "2026-05-14T00:00:05Z"
}
```

### 任务状态

`AiJobStatus`：

- `queued`
- `running`
- `succeeded`
- `failed`
- `cancelled`

`AiJobKind`：

- `chat`
- `image`
- `video`

`AiJobOperation`：

- `chat_completion`
- `text_to_image`
- `image_to_image`
- `text_to_video`
- `image_to_video`

第一阶段使用 `IAiJobStore` 边界和单例 `InMemoryAiJobStore`，不要把任务状态保存在 AppService 实例字段中。实现真实 Provider 前，`IAiJobStore` 可替换为数据库、分布式缓存或后台任务队列。

## Provider 抽象

Provider 接口建议按能力拆分：

```text
IAiChatProvider
IAiImageProvider
IAiVideoProvider
IAiProviderRouter
```

拆分原因：

- 供应商能力不一致，有些只支持图片，有些只支持视频。
- Chat 通常是同步或流式，媒体生成通常是异步任务。
- 后续可以让不同模型路由到不同服务，例如 chat 走 OpenAI，视频走 Runway 或 Vertex AI。

Provider Router 输入：

- `operation`
- `model`
- `tenantId`
- `appName`
- `userId`
- `metadata`

Provider Router 输出：

- Provider 名称。
- 实际模型名。
- Provider 实例。
- 是否支持同步返回、异步任务、multipart 输入和回调。

## 配置和密钥

第三方密钥只允许存在于服务端配置、环境变量或密钥管理系统中。客户端请求中不允许传递第三方 API Key。

建议配置形态：

```json
{
  "Ai": {
    "DefaultChatProvider": "Mock",
    "DefaultImageProvider": "Mock",
    "DefaultVideoProvider": "Mock",
    "Providers": {
      "Mock": {
        "Enabled": true
      }
    },
    "ModelAliases": {
      "puupee-chat": "mock-chat",
      "puupee-image": "mock-image",
      "puupee-video": "mock-video"
    }
  }
}
```

后续真实 Provider 增加自己的 `BaseUrl`、`ApiKey`、`TimeoutSeconds`、`DefaultModel` 和限流参数。

## 媒体输入和输出

客户端上传素材时，优先使用现有 `IStorageAppService` 获取对象存储上传凭证。AI 请求只传对象存储引用：

- `bucket`
- `key`
- `url`
- `mimeType`
- `width`
- `height`
- `sha256`

AI Gateway 从服务端读取或签名访问这些素材，再交给 Provider。生成结果由后端写回 Puupee 对象存储或保存外部 URL 引用，返回给客户端可预览的 `url` 和可同步的 `storageKey`。

第一阶段 Mock Provider 可以直接返回占位 URL，不实际上传文件。

## 安全和治理

所有 AI 接口默认需要登录态，除非后续有明确的内部 API Key 场景。

第一阶段需要保留这些安全边界：

- `[Authorize]` 保护 AppService 和 Controller。
- 不接受客户端传入第三方 API Key。
- 对 prompt、metadata 和文件引用做基础长度限制。
- 对 multipart 文件大小设置上限。
- 记录请求用户、应用名、模型别名、Provider 和任务 ID。
- 错误响应不暴露第三方密钥、完整上游请求头或供应商内部栈。

后续真实接入时增加：

- 用户额度和频率限制。
- 内容安全审核。
- 成本统计。
- 任务取消和超时回收。
- 供应商失败重试和降级策略。

## 小汪手工接入方式

小汪手工后续不直接调用第三方 AI。生成流程改为：

1. 用户选择款式图和布料图。
2. 客户端通过现有存储接口上传图片，得到 media 引用。
3. 调用 `POST /api/ai/images/edits` 生成成品效果图。
4. 调用 `POST /api/ai/images/generations` 或业务专用 prompt 生成裁剪图。
5. 调用 `POST /api/ai/videos/edits` 生成过程视频。
6. 轮询 `GET /api/ai/jobs/{id}`，把输出媒体写回 `HandcraftProject`。

裁剪图第一阶段可以仍由客户端或 Mock Provider 生成占位图。真实版本推荐后端增加一个更业务化的 `handcraft` 生成编排服务，内部调用通用 AI Gateway，但客户端仍只感知作品任务。

## 错误处理

统一错误字段：

- `code`
- `message`
- `provider`
- `retryable`
- `details`

典型错误：

- `invalid_prompt`
- `invalid_media`
- `unsupported_model`
- `provider_unavailable`
- `quota_exceeded`
- `content_rejected`
- `job_not_found`
- `job_cancelled`
- `generation_failed`

OpenAI 兼容接口返回 OpenAI 风格错误对象。Puupee 原生接口返回 ABP 现有错误外壳或统一 `AiErrorDto`，具体实现保持和当前 API 错误约定一致。

## 测试和验收

第一阶段需要测试：

- Chat DTO 能映射成 OpenAI 兼容响应。
- 文生图请求返回 `succeeded` 或可查询的任务对象。
- 图生图请求能接收媒体引用并返回图片输出。
- 文生视频请求返回异步任务。
- 图生视频请求能接收 prompt 和参考图引用。
- Job 查询能返回同一个任务 ID 的状态和输出。
- Mock Provider 不依赖外部网络和真实 API Key。
- 未登录请求被拒绝。
- 客户端请求中即使传入 `api_key`、`authorization` 之类字段，也不会被当作第三方密钥使用。

建议命令：

```bash
cd api
dotnet test
```

如果全量测试成本过高，先运行：

```bash
cd api
dotnet test test/Puupees.Application.Tests/Puupees.Application.Tests.csproj
```

验收标准：

- Swagger 能看到 Puupee 原生 AI 接口。
- OpenAI 兼容 Controller 的路由固定且不依赖 ABP conventional 命名。
- Mock Provider 可以让客户端完成 chat、图片、视频任务的端到端调用。
- 任何客户端代码都不需要配置第三方 AI API Key。

## 后续扩展

后续真实版本按 Provider 增量接入：

1. OpenAI Provider
   - Chat Completions 或 Responses。
   - Image Generations 和 Image Edits。
   - Video API。

2. Vertex AI Provider
   - Imagen 文生图和图生图。
   - Veo 文生视频和图生视频。

3. Runway Provider
   - 侧重视频生成和图生视频。

4. fal / Replicate Provider
   - 适合快速试验不同开源模型。

5. Self Hosted Provider
   - ComfyUI、Diffusers 或内部模型服务。

真实 Provider 接入时不改变客户端 DTO，只新增 Provider 实现、配置项、任务存储和必要的后台执行能力。
