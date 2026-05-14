# Puupee AI Gateway Implementation Plan（中文实施计划）

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 在 `api` 项目新增第一阶段 AI Gateway：统一 chat、文生图、图生图、文生视频、图生视频接口，提供 Puupee 原生 DTO 和 OpenAI 兼容外壳，并用 Mock Provider 打通端到端任务流。

**Architecture:** 使用 ABP 现有 `Application.Contracts` + `Application` + `HttpApi` 分层。Puupee 原生接口通过 AppService 承载业务契约，显式 MVC Controller 暴露固定 `/api/ai/**` 和 `/api/ai/v1/**` 路由；内部通过 Provider Router 分发到 Mock Provider，媒体任务通过 `IAiJobStore` 保存状态。

**Tech Stack:** .NET 8、ABP Framework 8.1、ASP.NET Core MVC、System.Text.Json、xUnit、Shouldly。

---

## 文件结构

第一阶段新增和修改这些文件：

- Create: `api/src/Puupees.Application.Contracts/Ai/IAiChatAppService.cs`
- Create: `api/src/Puupees.Application.Contracts/Ai/IAiImageAppService.cs`
- Create: `api/src/Puupees.Application.Contracts/Ai/IAiVideoAppService.cs`
- Create: `api/src/Puupees.Application.Contracts/Ai/IAiJobAppService.cs`
- Create: `api/src/Puupees.Application.Contracts/Ai/Dtos/AiEnums.cs`
- Create: `api/src/Puupees.Application.Contracts/Ai/Dtos/AiMediaDtos.cs`
- Create: `api/src/Puupees.Application.Contracts/Ai/Dtos/AiChatDtos.cs`
- Create: `api/src/Puupees.Application.Contracts/Ai/Dtos/AiImageDtos.cs`
- Create: `api/src/Puupees.Application.Contracts/Ai/Dtos/AiVideoDtos.cs`
- Create: `api/src/Puupees.Application.Contracts/Ai/Dtos/OpenAiCompatibleDtos.cs`
- Create: `api/src/Puupees.Application.Contracts/Ai/OpenAiCompatibleMappingExtensions.cs`
- Create: `api/src/Puupees.Application/Ai/AiChatAppService.cs`
- Create: `api/src/Puupees.Application/Ai/AiImageAppService.cs`
- Create: `api/src/Puupees.Application/Ai/AiVideoAppService.cs`
- Create: `api/src/Puupees.Application/Ai/AiJobAppService.cs`
- Create: `api/src/Puupees.Application/Ai/Jobs/IAiJobStore.cs`
- Create: `api/src/Puupees.Application/Ai/Jobs/InMemoryAiJobStore.cs`
- Create: `api/src/Puupees.Application/Ai/Providers/IAiProviderRouter.cs`
- Create: `api/src/Puupees.Application/Ai/Providers/IAiChatProvider.cs`
- Create: `api/src/Puupees.Application/Ai/Providers/IAiImageProvider.cs`
- Create: `api/src/Puupees.Application/Ai/Providers/IAiVideoProvider.cs`
- Create: `api/src/Puupees.Application/Ai/Providers/DefaultAiProviderRouter.cs`
- Create: `api/src/Puupees.Application/Ai/Providers/MockAiProvider.cs`
- Modify: `api/src/Puupees.Application/PuupeesApplicationModule.cs`
- Create: `api/src/Puupees.HttpApi/Controllers/Ai/AiChatController.cs`
- Create: `api/src/Puupees.HttpApi/Controllers/Ai/AiImagesController.cs`
- Create: `api/src/Puupees.HttpApi/Controllers/Ai/AiVideosController.cs`
- Create: `api/src/Puupees.HttpApi/Controllers/Ai/AiJobsController.cs`
- Create: `api/src/Puupees.HttpApi/Controllers/Ai/OpenAiCompatibleChatController.cs`
- Create: `api/src/Puupees.HttpApi/Controllers/Ai/OpenAiCompatibleImagesController.cs`
- Create: `api/src/Puupees.HttpApi/Controllers/Ai/OpenAiCompatibleVideosController.cs`
- Modify: `api/test/Puupees.Application.Tests/Puupees.Application.Tests.csproj`
- Create: `api/test/Puupees.Application.Tests/Ai/AiGatewayAppServiceTests.cs`
- Create: `api/test/Puupees.Application.Tests/Ai/OpenAiCompatibleMappingTests.cs`
- Create: `api/test/Puupees.Application.Tests/Ai/AiControllerRouteAndSecurityTests.cs`

## Task 1: 创建契约 DTO 和 AppService 接口

**Files:**
- Create: `api/src/Puupees.Application.Contracts/Ai/Dtos/AiEnums.cs`
- Create: `api/src/Puupees.Application.Contracts/Ai/Dtos/AiMediaDtos.cs`
- Create: `api/src/Puupees.Application.Contracts/Ai/Dtos/AiChatDtos.cs`
- Create: `api/src/Puupees.Application.Contracts/Ai/Dtos/AiImageDtos.cs`
- Create: `api/src/Puupees.Application.Contracts/Ai/Dtos/AiVideoDtos.cs`
- Create: `api/src/Puupees.Application.Contracts/Ai/Dtos/OpenAiCompatibleDtos.cs`
- Create: `api/src/Puupees.Application.Contracts/Ai/OpenAiCompatibleMappingExtensions.cs`
- Create: `api/src/Puupees.Application.Contracts/Ai/IAiChatAppService.cs`
- Create: `api/src/Puupees.Application.Contracts/Ai/IAiImageAppService.cs`
- Create: `api/src/Puupees.Application.Contracts/Ai/IAiVideoAppService.cs`
- Create: `api/src/Puupees.Application.Contracts/Ai/IAiJobAppService.cs`

- [ ] **Step 1: 创建枚举文件**

Create `api/src/Puupees.Application.Contracts/Ai/Dtos/AiEnums.cs`:

```csharp
using System.Text.Json.Serialization;

namespace Puupees.Ai.Dtos;

[JsonConverter(typeof(JsonStringEnumConverter))]
public enum AiJobStatus
{
    Queued,
    Running,
    Succeeded,
    Failed,
    Cancelled
}

[JsonConverter(typeof(JsonStringEnumConverter))]
public enum AiJobKind
{
    Chat,
    Image,
    Video
}

[JsonConverter(typeof(JsonStringEnumConverter))]
public enum AiJobOperation
{
    ChatCompletion,
    TextToImage,
    ImageToImage,
    TextToVideo,
    ImageToVideo
}

[JsonConverter(typeof(JsonStringEnumConverter))]
public enum AiMediaType
{
    Image,
    Video
}
```

- [ ] **Step 2: 创建媒体与任务 DTO**

Create `api/src/Puupees.Application.Contracts/Ai/Dtos/AiMediaDtos.cs`:

```csharp
using System;
using System.Collections.Generic;

namespace Puupees.Ai.Dtos;

[Serializable]
public class AiMediaReferenceDto
{
    public string? Bucket { get; set; }
    public string? Key { get; set; }
    public string? Url { get; set; }
    public string? MimeType { get; set; }
    public int? Width { get; set; }
    public int? Height { get; set; }
    public string? Sha256 { get; set; }
}

[Serializable]
public class AiMediaOutputDto
{
    public AiMediaType Type { get; set; }
    public string Url { get; set; } = string.Empty;
    public string? StorageKey { get; set; }
    public string? MimeType { get; set; }
    public int? Width { get; set; }
    public int? Height { get; set; }
    public double? DurationSeconds { get; set; }
}

[Serializable]
public class AiUsageDto
{
    public int? PromptTokens { get; set; }
    public int? CompletionTokens { get; set; }
    public int? TotalTokens { get; set; }
}

[Serializable]
public class AiErrorDto
{
    public string Code { get; set; } = string.Empty;
    public string Message { get; set; } = string.Empty;
    public string? Provider { get; set; }
    public bool Retryable { get; set; }
    public Dictionary<string, string> Details { get; set; } = new();
}

[Serializable]
public class AiJobDto
{
    public string Id { get; set; } = string.Empty;
    public AiJobKind Kind { get; set; }
    public AiJobOperation Operation { get; set; }
    public AiJobStatus Status { get; set; }
    public string Provider { get; set; } = string.Empty;
    public string Model { get; set; } = string.Empty;
    public string? Prompt { get; set; }
    public List<AiMediaOutputDto> Outputs { get; set; } = new();
    public AiUsageDto? Usage { get; set; }
    public AiErrorDto? Error { get; set; }
    public DateTime CreatedAt { get; set; }
    public DateTime UpdatedAt { get; set; }
    public DateTime? CompletedAt { get; set; }
}
```

- [ ] **Step 3: 创建 Chat DTO**

Create `api/src/Puupees.Application.Contracts/Ai/Dtos/AiChatDtos.cs`:

```csharp
using System;
using System.Collections.Generic;

namespace Puupees.Ai.Dtos;

[Serializable]
public class AiChatMessageDto
{
    public string Role { get; set; } = string.Empty;
    public string Content { get; set; } = string.Empty;
    public string? Name { get; set; }
}

[Serializable]
public class CreateAiChatCompletionDto
{
    public string Model { get; set; } = "puupee-chat";
    public List<AiChatMessageDto> Messages { get; set; } = new();
    public double? Temperature { get; set; }
    public double? TopP { get; set; }
    public int? MaxTokens { get; set; }
    public bool Stream { get; set; }
    public Dictionary<string, string> Metadata { get; set; } = new();
}

[Serializable]
public class AiChatChoiceDto
{
    public int Index { get; set; }
    public AiChatMessageDto Message { get; set; } = new();
    public string FinishReason { get; set; } = "stop";
}

[Serializable]
public class AiChatCompletionDto
{
    public string Id { get; set; } = string.Empty;
    public string Object { get; set; } = "chat.completion";
    public long Created { get; set; }
    public string Model { get; set; } = string.Empty;
    public List<AiChatChoiceDto> Choices { get; set; } = new();
    public AiUsageDto Usage { get; set; } = new();
}
```

- [ ] **Step 4: 创建图片和视频 DTO**

Create `api/src/Puupees.Application.Contracts/Ai/Dtos/AiImageDtos.cs`:

```csharp
using System;
using System.Collections.Generic;

namespace Puupees.Ai.Dtos;

[Serializable]
public class CreateAiImageGenerationDto
{
    public string Model { get; set; } = "puupee-image";
    public string Prompt { get; set; } = string.Empty;
    public string? Size { get; set; }
    public string? Quality { get; set; }
    public int N { get; set; } = 1;
    public Dictionary<string, string> Metadata { get; set; } = new();
}

[Serializable]
public class CreateAiImageEditDto
{
    public string Model { get; set; } = "puupee-image";
    public string Prompt { get; set; } = string.Empty;
    public List<AiMediaReferenceDto> InputImages { get; set; } = new();
    public AiMediaReferenceDto? Mask { get; set; }
    public string? Size { get; set; }
    public string? Quality { get; set; }
    public int N { get; set; } = 1;
    public Dictionary<string, string> Metadata { get; set; } = new();
}
```

Create `api/src/Puupees.Application.Contracts/Ai/Dtos/AiVideoDtos.cs`:

```csharp
using System;
using System.Collections.Generic;

namespace Puupees.Ai.Dtos;

[Serializable]
public class CreateAiVideoGenerationDto
{
    public string Model { get; set; } = "puupee-video";
    public string Prompt { get; set; } = string.Empty;
    public int? Width { get; set; }
    public int? Height { get; set; }
    public double? DurationSeconds { get; set; }
    public Dictionary<string, string> Metadata { get; set; } = new();
}

[Serializable]
public class CreateAiVideoEditDto
{
    public string Model { get; set; } = "puupee-video";
    public string Prompt { get; set; } = string.Empty;
    public List<AiMediaReferenceDto> InputImages { get; set; } = new();
    public int? Width { get; set; }
    public int? Height { get; set; }
    public double? DurationSeconds { get; set; }
    public Dictionary<string, string> Metadata { get; set; } = new();
}
```

- [ ] **Step 5: 创建 OpenAI 兼容 DTO 和映射**

Create `api/src/Puupees.Application.Contracts/Ai/Dtos/OpenAiCompatibleDtos.cs`:

```csharp
using System;
using System.Collections.Generic;
using System.Text.Json.Serialization;

namespace Puupees.Ai.Dtos;

[Serializable]
public class OpenAiChatCompletionRequestDto
{
    public string Model { get; set; } = "puupee-chat";
    public List<AiChatMessageDto> Messages { get; set; } = new();
    public double? Temperature { get; set; }

    [JsonPropertyName("top_p")]
    public double? TopP { get; set; }

    [JsonPropertyName("max_tokens")]
    public int? MaxTokens { get; set; }

    public bool Stream { get; set; }
    public Dictionary<string, string> Metadata { get; set; } = new();
}

[Serializable]
public class OpenAiImageGenerationRequestDto
{
    public string Model { get; set; } = "puupee-image";
    public string Prompt { get; set; } = string.Empty;
    public string? Size { get; set; }
    public string? Quality { get; set; }
    public int N { get; set; } = 1;

    [JsonPropertyName("response_format")]
    public string ResponseFormat { get; set; } = "url";

    public Dictionary<string, string> Metadata { get; set; } = new();
}

[Serializable]
public class OpenAiImageDataDto
{
    public string? Url { get; set; }

    [JsonPropertyName("b64_json")]
    public string? B64Json { get; set; }

    [JsonPropertyName("revised_prompt")]
    public string? RevisedPrompt { get; set; }
}

[Serializable]
public class OpenAiImageResponseDto
{
    public long Created { get; set; }
    public List<OpenAiImageDataDto> Data { get; set; } = new();
}

[Serializable]
public class OpenAiVideoRequestDto
{
    public string Model { get; set; } = "puupee-video";
    public string Prompt { get; set; } = string.Empty;

    [JsonPropertyName("input_images")]
    public List<AiMediaReferenceDto> InputImages { get; set; } = new();

    public int? Width { get; set; }
    public int? Height { get; set; }

    [JsonPropertyName("duration_seconds")]
    public double? DurationSeconds { get; set; }

    public Dictionary<string, string> Metadata { get; set; } = new();
}

[Serializable]
public class OpenAiVideoResponseDto
{
    public string Id { get; set; } = string.Empty;
    public string Object { get; set; } = "video.generation";
    public string Status { get; set; } = string.Empty;
    public string Model { get; set; } = string.Empty;

    [JsonPropertyName("created_at")]
    public long CreatedAt { get; set; }

    [JsonPropertyName("completed_at")]
    public long? CompletedAt { get; set; }

    public AiErrorDto? Error { get; set; }
    public List<AiMediaOutputDto> Outputs { get; set; } = new();
}
```

Create `api/src/Puupees.Application.Contracts/Ai/OpenAiCompatibleMappingExtensions.cs`:

```csharp
using System;
using System.Linq;
using Puupees.Ai.Dtos;

namespace Puupees.Ai;

public static class OpenAiCompatibleMappingExtensions
{
    public static CreateAiChatCompletionDto ToPuupeeInput(this OpenAiChatCompletionRequestDto input)
    {
        return new CreateAiChatCompletionDto
        {
            Model = input.Model,
            Messages = input.Messages,
            Temperature = input.Temperature,
            TopP = input.TopP,
            MaxTokens = input.MaxTokens,
            Stream = input.Stream,
            Metadata = input.Metadata
        };
    }

    public static CreateAiImageGenerationDto ToPuupeeInput(this OpenAiImageGenerationRequestDto input)
    {
        return new CreateAiImageGenerationDto
        {
            Model = input.Model,
            Prompt = input.Prompt,
            Size = input.Size,
            Quality = input.Quality,
            N = input.N,
            Metadata = input.Metadata
        };
    }

    public static OpenAiImageResponseDto ToOpenAiImageResponse(this AiJobDto job)
    {
        return new OpenAiImageResponseDto
        {
            Created = ToUnixTimeSeconds(job.CreatedAt),
            Data = job.Outputs.Select(output => new OpenAiImageDataDto
            {
                Url = output.Url,
                RevisedPrompt = job.Prompt
            }).ToList()
        };
    }

    public static OpenAiVideoResponseDto ToOpenAiVideoResponse(this AiJobDto job)
    {
        return new OpenAiVideoResponseDto
        {
            Id = job.Id,
            Status = ToOpenAiStatus(job.Status),
            Model = job.Model,
            CreatedAt = ToUnixTimeSeconds(job.CreatedAt),
            CompletedAt = job.CompletedAt.HasValue ? ToUnixTimeSeconds(job.CompletedAt.Value) : null,
            Error = job.Error,
            Outputs = job.Outputs
        };
    }

    public static long ToUnixTimeSeconds(DateTime value)
    {
        var utc = value.Kind == DateTimeKind.Utc ? value : DateTime.SpecifyKind(value, DateTimeKind.Utc);
        return new DateTimeOffset(utc).ToUnixTimeSeconds();
    }

    public static string ToOpenAiStatus(AiJobStatus status)
    {
        return status switch
        {
            AiJobStatus.Queued => "queued",
            AiJobStatus.Running => "running",
            AiJobStatus.Succeeded => "succeeded",
            AiJobStatus.Failed => "failed",
            AiJobStatus.Cancelled => "cancelled",
            _ => "failed"
        };
    }
}
```

- [ ] **Step 6: 创建 AppService 接口**

Create `api/src/Puupees.Application.Contracts/Ai/IAiChatAppService.cs`:

```csharp
using System.Threading.Tasks;
using Puupees.Ai.Dtos;
using Volo.Abp.Application.Services;

namespace Puupees.Ai;

public interface IAiChatAppService : IApplicationService
{
    Task<AiChatCompletionDto> CreateCompletionAsync(CreateAiChatCompletionDto input);
}
```

Create `api/src/Puupees.Application.Contracts/Ai/IAiImageAppService.cs`:

```csharp
using System.Threading.Tasks;
using Puupees.Ai.Dtos;
using Volo.Abp.Application.Services;

namespace Puupees.Ai;

public interface IAiImageAppService : IApplicationService
{
    Task<AiJobDto> CreateGenerationAsync(CreateAiImageGenerationDto input);
    Task<AiJobDto> CreateEditAsync(CreateAiImageEditDto input);
}
```

Create `api/src/Puupees.Application.Contracts/Ai/IAiVideoAppService.cs`:

```csharp
using System.Threading.Tasks;
using Puupees.Ai.Dtos;
using Volo.Abp.Application.Services;

namespace Puupees.Ai;

public interface IAiVideoAppService : IApplicationService
{
    Task<AiJobDto> CreateGenerationAsync(CreateAiVideoGenerationDto input);
    Task<AiJobDto> CreateEditAsync(CreateAiVideoEditDto input);
}
```

Create `api/src/Puupees.Application.Contracts/Ai/IAiJobAppService.cs`:

```csharp
using System.Threading.Tasks;
using Puupees.Ai.Dtos;
using Volo.Abp.Application.Services;

namespace Puupees.Ai;

public interface IAiJobAppService : IApplicationService
{
    Task<AiJobDto> GetAsync(string id);
    Task<AiJobDto> CancelAsync(string id);
}
```

- [ ] **Step 7: 验证 Contracts 项目能编译**

Run:

```bash
cd api
dotnet build src/Puupees.Application.Contracts/Puupees.Application.Contracts.csproj
```

Expected: `Build succeeded.`，没有 `CS0246`、`CS8618` 或 JSON 属性命名相关错误。

- [ ] **Step 8: 提交契约**

```bash
git add api/src/Puupees.Application.Contracts/Ai
git commit -m "feat(api): 添加 AI Gateway 契约"
```

## Task 2: 先写 AppService 失败测试

**Files:**
- Create: `api/test/Puupees.Application.Tests/Ai/AiGatewayAppServiceTests.cs`

- [ ] **Step 1: 写 AppService 行为测试**

Create `api/test/Puupees.Application.Tests/Ai/AiGatewayAppServiceTests.cs`:

```csharp
using System.Threading.Tasks;
using Puupees.Ai;
using Puupees.Ai.Dtos;
using Shouldly;
using Xunit;

namespace Puupees.Tests.Ai;

public class AiGatewayAppServiceTests : PuupeesApplicationTestBase<PuupeesApplicationTestModule>
{
    private readonly IAiChatAppService _chatAppService;
    private readonly IAiImageAppService _imageAppService;
    private readonly IAiVideoAppService _videoAppService;
    private readonly IAiJobAppService _jobAppService;

    public AiGatewayAppServiceTests()
    {
        _chatAppService = GetRequiredService<IAiChatAppService>();
        _imageAppService = GetRequiredService<IAiImageAppService>();
        _videoAppService = GetRequiredService<IAiVideoAppService>();
        _jobAppService = GetRequiredService<IAiJobAppService>();
    }

    [Fact]
    public async Task Should_Create_Mock_Chat_Completion()
    {
        var result = await _chatAppService.CreateCompletionAsync(new CreateAiChatCompletionDto
        {
            Model = "puupee-chat",
            Messages =
            {
                new AiChatMessageDto { Role = "user", Content = "你好" }
            }
        });

        result.Id.ShouldStartWith("chatcmpl_mock_");
        result.Object.ShouldBe("chat.completion");
        result.Model.ShouldBe("mock-chat");
        result.Choices.Count.ShouldBe(1);
        result.Choices[0].Message.Role.ShouldBe("assistant");
        result.Choices[0].Message.Content.ShouldContain("Mock AI Gateway");
        result.Usage.TotalTokens.ShouldBeGreaterThan(0);
    }

    [Fact]
    public async Task Should_Create_Text_To_Image_Job_And_Query_It()
    {
        var job = await _imageAppService.CreateGenerationAsync(new CreateAiImageGenerationDto
        {
            Model = "puupee-image",
            Prompt = "生成一个蓝色托特包效果图",
            Size = "1024x1024",
            N = 1
        });

        job.Id.ShouldStartWith("ai_job_mock_");
        job.Kind.ShouldBe(AiJobKind.Image);
        job.Operation.ShouldBe(AiJobOperation.TextToImage);
        job.Status.ShouldBe(AiJobStatus.Succeeded);
        job.Provider.ShouldBe("mock");
        job.Model.ShouldBe("mock-image");
        job.Outputs.Count.ShouldBe(1);
        job.Outputs[0].Type.ShouldBe(AiMediaType.Image);
        job.Outputs[0].Url.ShouldContain("text-to-image");

        var queried = await _jobAppService.GetAsync(job.Id);
        queried.Id.ShouldBe(job.Id);
        queried.Status.ShouldBe(AiJobStatus.Succeeded);
    }

    [Fact]
    public async Task Should_Create_Image_To_Image_Job()
    {
        var job = await _imageAppService.CreateEditAsync(new CreateAiImageEditDto
        {
            Model = "puupee-image",
            Prompt = "使用牛仔布重绘这个包",
            InputImages =
            {
                new AiMediaReferenceDto
                {
                    Bucket = "puupee-test",
                    Key = "users/test/style.png",
                    MimeType = "image/png"
                },
                new AiMediaReferenceDto
                {
                    Bucket = "puupee-test",
                    Key = "users/test/fabric.png",
                    MimeType = "image/png"
                }
            }
        });

        job.Kind.ShouldBe(AiJobKind.Image);
        job.Operation.ShouldBe(AiJobOperation.ImageToImage);
        job.Status.ShouldBe(AiJobStatus.Succeeded);
        job.Outputs.Count.ShouldBe(1);
        job.Outputs[0].Url.ShouldContain("image-to-image");
    }

    [Fact]
    public async Task Should_Create_Text_To_Video_Job()
    {
        var job = await _videoAppService.CreateGenerationAsync(new CreateAiVideoGenerationDto
        {
            Model = "puupee-video",
            Prompt = "展示布料被裁剪和缝纫成包的过程",
            DurationSeconds = 8
        });

        job.Kind.ShouldBe(AiJobKind.Video);
        job.Operation.ShouldBe(AiJobOperation.TextToVideo);
        job.Status.ShouldBe(AiJobStatus.Succeeded);
        job.Outputs.Count.ShouldBe(1);
        job.Outputs[0].Type.ShouldBe(AiMediaType.Video);
        job.Outputs[0].Url.ShouldContain("text-to-video");
    }

    [Fact]
    public async Task Should_Create_Image_To_Video_Job()
    {
        var job = await _videoAppService.CreateEditAsync(new CreateAiVideoEditDto
        {
            Model = "puupee-video",
            Prompt = "从布料到成品的缝纫过程",
            InputImages =
            {
                new AiMediaReferenceDto
                {
                    Bucket = "puupee-test",
                    Key = "users/test/fabric.png",
                    MimeType = "image/png"
                }
            },
            DurationSeconds = 8
        });

        job.Kind.ShouldBe(AiJobKind.Video);
        job.Operation.ShouldBe(AiJobOperation.ImageToVideo);
        job.Status.ShouldBe(AiJobStatus.Succeeded);
        job.Outputs.Count.ShouldBe(1);
        job.Outputs[0].Type.ShouldBe(AiMediaType.Video);
        job.Outputs[0].Url.ShouldContain("image-to-video");
    }
}
```

- [ ] **Step 2: 运行测试并确认失败原因正确**

Run:

```bash
cd api
dotnet test test/Puupees.Application.Tests/Puupees.Application.Tests.csproj --filter FullyQualifiedName~AiGatewayAppServiceTests
```

Expected: FAIL，失败原因是 `IAiChatAppService`、`IAiImageAppService`、`IAiVideoAppService` 或 `IAiJobAppService` 尚未注册或实现。不能出现 Contracts 编译错误。

- [ ] **Step 3: 提交失败测试**

```bash
git add api/test/Puupees.Application.Tests/Ai/AiGatewayAppServiceTests.cs
git commit -m "test(api): 添加 AI Gateway 应用服务测试"
```

## Task 3: 实现 Job Store、Provider Router 和 Mock Provider

**Files:**
- Create: `api/src/Puupees.Application/Ai/Jobs/IAiJobStore.cs`
- Create: `api/src/Puupees.Application/Ai/Jobs/InMemoryAiJobStore.cs`
- Create: `api/src/Puupees.Application/Ai/Providers/IAiProviderRouter.cs`
- Create: `api/src/Puupees.Application/Ai/Providers/IAiChatProvider.cs`
- Create: `api/src/Puupees.Application/Ai/Providers/IAiImageProvider.cs`
- Create: `api/src/Puupees.Application/Ai/Providers/IAiVideoProvider.cs`
- Create: `api/src/Puupees.Application/Ai/Providers/DefaultAiProviderRouter.cs`
- Create: `api/src/Puupees.Application/Ai/Providers/MockAiProvider.cs`
- Modify: `api/src/Puupees.Application/PuupeesApplicationModule.cs`

- [ ] **Step 1: 创建 Job Store 接口**

Create `api/src/Puupees.Application/Ai/Jobs/IAiJobStore.cs`:

```csharp
using System.Threading.Tasks;
using Puupees.Ai.Dtos;

namespace Puupees.Ai.Jobs;

public interface IAiJobStore
{
    Task<AiJobDto> SaveAsync(AiJobDto job);
    Task<AiJobDto?> FindAsync(string id);
    Task<AiJobDto> CancelAsync(string id);
}
```

- [ ] **Step 2: 创建内存 Job Store**

Create `api/src/Puupees.Application/Ai/Jobs/InMemoryAiJobStore.cs`:

```csharp
using System;
using System.Collections.Concurrent;
using System.Threading.Tasks;
using Puupees.Ai.Dtos;
using Volo.Abp;

namespace Puupees.Ai.Jobs;

public class InMemoryAiJobStore : IAiJobStore
{
    private readonly ConcurrentDictionary<string, AiJobDto> _jobs = new();

    public Task<AiJobDto> SaveAsync(AiJobDto job)
    {
        _jobs[job.Id] = job;
        return Task.FromResult(job);
    }

    public Task<AiJobDto?> FindAsync(string id)
    {
        _jobs.TryGetValue(id, out var job);
        return Task.FromResult(job);
    }

    public Task<AiJobDto> CancelAsync(string id)
    {
        if (!_jobs.TryGetValue(id, out var job))
        {
            throw new BusinessException("Puupees.Ai:JobNotFound")
                .WithData("Id", id);
        }

        if (job.Status is AiJobStatus.Queued or AiJobStatus.Running)
        {
            job.Status = AiJobStatus.Cancelled;
            job.UpdatedAt = DateTime.UtcNow;
            job.CompletedAt = job.UpdatedAt;
            _jobs[id] = job;
        }

        return Task.FromResult(job);
    }
}
```

- [ ] **Step 3: 创建 Provider 接口和 Router**

Create `api/src/Puupees.Application/Ai/Providers/IAiChatProvider.cs`:

```csharp
using System.Threading.Tasks;
using Puupees.Ai.Dtos;

namespace Puupees.Ai.Providers;

public interface IAiChatProvider
{
    string Name { get; }
    Task<AiChatCompletionDto> CreateCompletionAsync(CreateAiChatCompletionDto input);
}
```

Create `api/src/Puupees.Application/Ai/Providers/IAiImageProvider.cs`:

```csharp
using System.Threading.Tasks;
using Puupees.Ai.Dtos;

namespace Puupees.Ai.Providers;

public interface IAiImageProvider
{
    string Name { get; }
    Task<AiJobDto> CreateGenerationAsync(CreateAiImageGenerationDto input);
    Task<AiJobDto> CreateEditAsync(CreateAiImageEditDto input);
}
```

Create `api/src/Puupees.Application/Ai/Providers/IAiVideoProvider.cs`:

```csharp
using System.Threading.Tasks;
using Puupees.Ai.Dtos;

namespace Puupees.Ai.Providers;

public interface IAiVideoProvider
{
    string Name { get; }
    Task<AiJobDto> CreateGenerationAsync(CreateAiVideoGenerationDto input);
    Task<AiJobDto> CreateEditAsync(CreateAiVideoEditDto input);
}
```

Create `api/src/Puupees.Application/Ai/Providers/IAiProviderRouter.cs`:

```csharp
using System.Threading.Tasks;

namespace Puupees.Ai.Providers;

public interface IAiProviderRouter
{
    Task<IAiChatProvider> GetChatProviderAsync(string model);
    Task<IAiImageProvider> GetImageProviderAsync(string model);
    Task<IAiVideoProvider> GetVideoProviderAsync(string model);
}
```

Create `api/src/Puupees.Application/Ai/Providers/DefaultAiProviderRouter.cs`:

```csharp
using System.Threading.Tasks;

namespace Puupees.Ai.Providers;

public class DefaultAiProviderRouter : IAiProviderRouter
{
    private readonly MockAiProvider _mockAiProvider;

    public DefaultAiProviderRouter(MockAiProvider mockAiProvider)
    {
        _mockAiProvider = mockAiProvider;
    }

    public Task<IAiChatProvider> GetChatProviderAsync(string model)
    {
        return Task.FromResult<IAiChatProvider>(_mockAiProvider);
    }

    public Task<IAiImageProvider> GetImageProviderAsync(string model)
    {
        return Task.FromResult<IAiImageProvider>(_mockAiProvider);
    }

    public Task<IAiVideoProvider> GetVideoProviderAsync(string model)
    {
        return Task.FromResult<IAiVideoProvider>(_mockAiProvider);
    }
}
```

- [ ] **Step 4: 创建 Mock Provider**

Create `api/src/Puupees.Application/Ai/Providers/MockAiProvider.cs`:

```csharp
using System;
using System.Collections.Generic;
using System.Linq;
using System.Threading.Tasks;
using Puupees.Ai.Dtos;

namespace Puupees.Ai.Providers;

public class MockAiProvider : IAiChatProvider, IAiImageProvider, IAiVideoProvider
{
    public string Name => "mock";

    public Task<AiChatCompletionDto> CreateCompletionAsync(CreateAiChatCompletionDto input)
    {
        var created = DateTimeOffset.UtcNow.ToUnixTimeSeconds();
        var userText = input.Messages.LastOrDefault(m => m.Role == "user")?.Content ?? string.Empty;

        return Task.FromResult(new AiChatCompletionDto
        {
            Id = $"chatcmpl_mock_{Guid.NewGuid():N}",
            Created = created,
            Model = "mock-chat",
            Choices =
            {
                new AiChatChoiceDto
                {
                    Index = 0,
                    Message = new AiChatMessageDto
                    {
                        Role = "assistant",
                        Content = $"Mock AI Gateway response: {userText}"
                    },
                    FinishReason = "stop"
                }
            },
            Usage = new AiUsageDto
            {
                PromptTokens = Math.Max(1, userText.Length / 2),
                CompletionTokens = 8,
                TotalTokens = Math.Max(1, userText.Length / 2) + 8
            }
        });
    }

    public Task<AiJobDto> CreateGenerationAsync(CreateAiImageGenerationDto input)
    {
        return Task.FromResult(CreateSucceededJob(
            AiJobKind.Image,
            AiJobOperation.TextToImage,
            "mock-image",
            input.Prompt,
            new AiMediaOutputDto
            {
                Type = AiMediaType.Image,
                Url = "https://cdn.puupee.com/ai/mock/text-to-image.png",
                StorageKey = "ai/mock/text-to-image.png",
                MimeType = "image/png",
                Width = 1024,
                Height = 1024
            }));
    }

    public Task<AiJobDto> CreateEditAsync(CreateAiImageEditDto input)
    {
        return Task.FromResult(CreateSucceededJob(
            AiJobKind.Image,
            AiJobOperation.ImageToImage,
            "mock-image",
            input.Prompt,
            new AiMediaOutputDto
            {
                Type = AiMediaType.Image,
                Url = "https://cdn.puupee.com/ai/mock/image-to-image.png",
                StorageKey = "ai/mock/image-to-image.png",
                MimeType = "image/png",
                Width = 1024,
                Height = 1024
            }));
    }

    public Task<AiJobDto> CreateGenerationAsync(CreateAiVideoGenerationDto input)
    {
        return Task.FromResult(CreateSucceededJob(
            AiJobKind.Video,
            AiJobOperation.TextToVideo,
            "mock-video",
            input.Prompt,
            new AiMediaOutputDto
            {
                Type = AiMediaType.Video,
                Url = "https://cdn.puupee.com/ai/mock/text-to-video.mp4",
                StorageKey = "ai/mock/text-to-video.mp4",
                MimeType = "video/mp4",
                Width = input.Width ?? 1280,
                Height = input.Height ?? 720,
                DurationSeconds = input.DurationSeconds ?? 8
            }));
    }

    public Task<AiJobDto> CreateEditAsync(CreateAiVideoEditDto input)
    {
        return Task.FromResult(CreateSucceededJob(
            AiJobKind.Video,
            AiJobOperation.ImageToVideo,
            "mock-video",
            input.Prompt,
            new AiMediaOutputDto
            {
                Type = AiMediaType.Video,
                Url = "https://cdn.puupee.com/ai/mock/image-to-video.mp4",
                StorageKey = "ai/mock/image-to-video.mp4",
                MimeType = "video/mp4",
                Width = input.Width ?? 1280,
                Height = input.Height ?? 720,
                DurationSeconds = input.DurationSeconds ?? 8
            }));
    }

    private static AiJobDto CreateSucceededJob(
        AiJobKind kind,
        AiJobOperation operation,
        string model,
        string prompt,
        AiMediaOutputDto output)
    {
        var now = DateTime.UtcNow;

        return new AiJobDto
        {
            Id = $"ai_job_mock_{Guid.NewGuid():N}",
            Kind = kind,
            Operation = operation,
            Status = AiJobStatus.Succeeded,
            Provider = "mock",
            Model = model,
            Prompt = prompt,
            Outputs = new List<AiMediaOutputDto> { output },
            CreatedAt = now,
            UpdatedAt = now,
            CompletedAt = now
        };
    }
}
```

- [ ] **Step 5: 注册 AI 服务**

Modify `api/src/Puupees.Application/PuupeesApplicationModule.cs` inside `ConfigureServices` after existing service registrations:

```csharp
context.Services.AddSingleton<Puupees.Ai.Jobs.IAiJobStore, Puupees.Ai.Jobs.InMemoryAiJobStore>();
context.Services.AddSingleton<Puupees.Ai.Providers.MockAiProvider>();
context.Services.AddSingleton<Puupees.Ai.Providers.IAiProviderRouter, Puupees.Ai.Providers.DefaultAiProviderRouter>();
```

If the file does not currently include these namespaces, use fully qualified names as shown above so no extra `using` lines are required.

- [ ] **Step 6: 构建 Application 项目**

Run:

```bash
cd api
dotnet build src/Puupees.Application/Puupees.Application.csproj
```

Expected: `Build succeeded.`，测试仍然失败，因为 AppService 实现还没有创建。

- [ ] **Step 7: 提交 Provider 和 Job Store**

```bash
git add api/src/Puupees.Application/Ai/Jobs api/src/Puupees.Application/Ai/Providers api/src/Puupees.Application/PuupeesApplicationModule.cs
git commit -m "feat(api): 添加 AI Gateway Mock Provider"
```

## Task 4: 实现 Puupee 原生 AppService

**Files:**
- Create: `api/src/Puupees.Application/Ai/AiChatAppService.cs`
- Create: `api/src/Puupees.Application/Ai/AiImageAppService.cs`
- Create: `api/src/Puupees.Application/Ai/AiVideoAppService.cs`
- Create: `api/src/Puupees.Application/Ai/AiJobAppService.cs`

- [ ] **Step 1: 创建 Chat AppService**

Create `api/src/Puupees.Application/Ai/AiChatAppService.cs`:

```csharp
using System.Threading.Tasks;
using Microsoft.AspNetCore.Authorization;
using Puupees.Ai.Dtos;
using Puupees.Ai.Providers;
using Volo.Abp;

namespace Puupees.Ai;

[Authorize]
public class AiChatAppService : PuupeesAppServiceBase, IAiChatAppService
{
    private readonly IAiProviderRouter _providerRouter;

    public AiChatAppService(IAiProviderRouter providerRouter)
    {
        _providerRouter = providerRouter;
    }

    public async Task<AiChatCompletionDto> CreateCompletionAsync(CreateAiChatCompletionDto input)
    {
        if (input.Messages.Count == 0)
        {
            throw new BusinessException("Puupees.Ai:MessagesRequired");
        }

        var provider = await _providerRouter.GetChatProviderAsync(input.Model);
        return await provider.CreateCompletionAsync(input);
    }
}
```

- [ ] **Step 2: 创建 Image AppService**

Create `api/src/Puupees.Application/Ai/AiImageAppService.cs`:

```csharp
using System.Threading.Tasks;
using Microsoft.AspNetCore.Authorization;
using Puupees.Ai.Dtos;
using Puupees.Ai.Jobs;
using Puupees.Ai.Providers;
using Volo.Abp;

namespace Puupees.Ai;

[Authorize]
public class AiImageAppService : PuupeesAppServiceBase, IAiImageAppService
{
    private readonly IAiProviderRouter _providerRouter;
    private readonly IAiJobStore _jobStore;

    public AiImageAppService(IAiProviderRouter providerRouter, IAiJobStore jobStore)
    {
        _providerRouter = providerRouter;
        _jobStore = jobStore;
    }

    public async Task<AiJobDto> CreateGenerationAsync(CreateAiImageGenerationDto input)
    {
        if (string.IsNullOrWhiteSpace(input.Prompt))
        {
            throw new BusinessException("Puupees.Ai:PromptRequired");
        }

        var provider = await _providerRouter.GetImageProviderAsync(input.Model);
        var job = await provider.CreateGenerationAsync(input);
        return await _jobStore.SaveAsync(job);
    }

    public async Task<AiJobDto> CreateEditAsync(CreateAiImageEditDto input)
    {
        if (string.IsNullOrWhiteSpace(input.Prompt))
        {
            throw new BusinessException("Puupees.Ai:PromptRequired");
        }

        if (input.InputImages.Count == 0)
        {
            throw new BusinessException("Puupees.Ai:InputImagesRequired");
        }

        var provider = await _providerRouter.GetImageProviderAsync(input.Model);
        var job = await provider.CreateEditAsync(input);
        return await _jobStore.SaveAsync(job);
    }
}
```

- [ ] **Step 3: 创建 Video AppService**

Create `api/src/Puupees.Application/Ai/AiVideoAppService.cs`:

```csharp
using System.Threading.Tasks;
using Microsoft.AspNetCore.Authorization;
using Puupees.Ai.Dtos;
using Puupees.Ai.Jobs;
using Puupees.Ai.Providers;
using Volo.Abp;

namespace Puupees.Ai;

[Authorize]
public class AiVideoAppService : PuupeesAppServiceBase, IAiVideoAppService
{
    private readonly IAiProviderRouter _providerRouter;
    private readonly IAiJobStore _jobStore;

    public AiVideoAppService(IAiProviderRouter providerRouter, IAiJobStore jobStore)
    {
        _providerRouter = providerRouter;
        _jobStore = jobStore;
    }

    public async Task<AiJobDto> CreateGenerationAsync(CreateAiVideoGenerationDto input)
    {
        if (string.IsNullOrWhiteSpace(input.Prompt))
        {
            throw new BusinessException("Puupees.Ai:PromptRequired");
        }

        var provider = await _providerRouter.GetVideoProviderAsync(input.Model);
        var job = await provider.CreateGenerationAsync(input);
        return await _jobStore.SaveAsync(job);
    }

    public async Task<AiJobDto> CreateEditAsync(CreateAiVideoEditDto input)
    {
        if (string.IsNullOrWhiteSpace(input.Prompt))
        {
            throw new BusinessException("Puupees.Ai:PromptRequired");
        }

        if (input.InputImages.Count == 0)
        {
            throw new BusinessException("Puupees.Ai:InputImagesRequired");
        }

        var provider = await _providerRouter.GetVideoProviderAsync(input.Model);
        var job = await provider.CreateEditAsync(input);
        return await _jobStore.SaveAsync(job);
    }
}
```

- [ ] **Step 4: 创建 Job AppService**

Create `api/src/Puupees.Application/Ai/AiJobAppService.cs`:

```csharp
using System.Threading.Tasks;
using Microsoft.AspNetCore.Authorization;
using Puupees.Ai.Dtos;
using Puupees.Ai.Jobs;
using Volo.Abp;

namespace Puupees.Ai;

[Authorize]
public class AiJobAppService : PuupeesAppServiceBase, IAiJobAppService
{
    private readonly IAiJobStore _jobStore;

    public AiJobAppService(IAiJobStore jobStore)
    {
        _jobStore = jobStore;
    }

    public async Task<AiJobDto> GetAsync(string id)
    {
        var job = await _jobStore.FindAsync(id);
        if (job == null)
        {
            throw new BusinessException("Puupees.Ai:JobNotFound")
                .WithData("Id", id);
        }

        return job;
    }

    public Task<AiJobDto> CancelAsync(string id)
    {
        return _jobStore.CancelAsync(id);
    }
}
```

- [ ] **Step 5: 运行 AppService 测试**

Run:

```bash
cd api
dotnet test test/Puupees.Application.Tests/Puupees.Application.Tests.csproj --filter FullyQualifiedName~AiGatewayAppServiceTests
```

Expected: PASS，5 个 AI Gateway AppService 测试通过。

- [ ] **Step 6: 提交 AppService 实现**

```bash
git add api/src/Puupees.Application/Ai api/test/Puupees.Application.Tests/Ai/AiGatewayAppServiceTests.cs
git commit -m "feat(api): 实现 AI Gateway 应用服务"
```

## Task 5: 添加 OpenAI 兼容映射测试

**Files:**
- Create: `api/test/Puupees.Application.Tests/Ai/OpenAiCompatibleMappingTests.cs`

- [ ] **Step 1: 写映射测试**

Create `api/test/Puupees.Application.Tests/Ai/OpenAiCompatibleMappingTests.cs`:

```csharp
using System;
using Puupees.Ai;
using Puupees.Ai.Dtos;
using Shouldly;
using Xunit;

namespace Puupees.Tests.Ai;

public class OpenAiCompatibleMappingTests
{
    [Fact]
    public void Should_Map_OpenAi_Chat_Request_To_Puupee_Input()
    {
        var request = new OpenAiChatCompletionRequestDto
        {
            Model = "gpt-compatible",
            Messages =
            {
                new AiChatMessageDto { Role = "user", Content = "hello" }
            },
            Temperature = 0.2,
            TopP = 0.9,
            MaxTokens = 256,
            Stream = true
        };

        var input = request.ToPuupeeInput();

        input.Model.ShouldBe("gpt-compatible");
        input.Messages.Count.ShouldBe(1);
        input.Messages[0].Content.ShouldBe("hello");
        input.Temperature.ShouldBe(0.2);
        input.TopP.ShouldBe(0.9);
        input.MaxTokens.ShouldBe(256);
        input.Stream.ShouldBeTrue();
    }

    [Fact]
    public void Should_Map_Image_Job_To_OpenAi_Image_Response()
    {
        var job = new AiJobDto
        {
            Id = "ai_job_mock_test",
            Kind = AiJobKind.Image,
            Operation = AiJobOperation.TextToImage,
            Status = AiJobStatus.Succeeded,
            Provider = "mock",
            Model = "mock-image",
            Prompt = "blue bag",
            CreatedAt = new DateTime(2026, 5, 14, 0, 0, 0, DateTimeKind.Utc),
            UpdatedAt = new DateTime(2026, 5, 14, 0, 0, 0, DateTimeKind.Utc),
            Outputs =
            {
                new AiMediaOutputDto
                {
                    Type = AiMediaType.Image,
                    Url = "https://cdn.puupee.com/ai/mock/text-to-image.png"
                }
            }
        };

        var response = job.ToOpenAiImageResponse();

        response.Created.ShouldBeGreaterThan(0);
        response.Data.Count.ShouldBe(1);
        response.Data[0].Url.ShouldBe("https://cdn.puupee.com/ai/mock/text-to-image.png");
        response.Data[0].RevisedPrompt.ShouldBe("blue bag");
    }

    [Fact]
    public void Should_Map_Video_Job_To_OpenAi_Video_Response()
    {
        var job = new AiJobDto
        {
            Id = "ai_job_mock_video",
            Kind = AiJobKind.Video,
            Operation = AiJobOperation.ImageToVideo,
            Status = AiJobStatus.Succeeded,
            Provider = "mock",
            Model = "mock-video",
            Prompt = "make video",
            CreatedAt = new DateTime(2026, 5, 14, 0, 0, 0, DateTimeKind.Utc),
            UpdatedAt = new DateTime(2026, 5, 14, 0, 0, 1, DateTimeKind.Utc),
            CompletedAt = new DateTime(2026, 5, 14, 0, 0, 1, DateTimeKind.Utc)
        };

        var response = job.ToOpenAiVideoResponse();

        response.Id.ShouldBe("ai_job_mock_video");
        response.Object.ShouldBe("video.generation");
        response.Status.ShouldBe("succeeded");
        response.Model.ShouldBe("mock-video");
        response.CreatedAt.ShouldBeGreaterThan(0);
        response.CompletedAt.ShouldNotBeNull();
    }
}
```

- [ ] **Step 2: 运行映射测试**

Run:

```bash
cd api
dotnet test test/Puupees.Application.Tests/Puupees.Application.Tests.csproj --filter FullyQualifiedName~OpenAiCompatibleMappingTests
```

Expected: PASS，3 个映射测试通过。

- [ ] **Step 3: 提交映射测试**

```bash
git add api/test/Puupees.Application.Tests/Ai/OpenAiCompatibleMappingTests.cs
git commit -m "test(api): 覆盖 OpenAI 兼容映射"
```

## Task 6: 添加 Puupee 原生固定路由 Controller

**Files:**
- Create: `api/src/Puupees.HttpApi/Controllers/Ai/AiChatController.cs`
- Create: `api/src/Puupees.HttpApi/Controllers/Ai/AiImagesController.cs`
- Create: `api/src/Puupees.HttpApi/Controllers/Ai/AiVideosController.cs`
- Create: `api/src/Puupees.HttpApi/Controllers/Ai/AiJobsController.cs`

- [ ] **Step 1: 创建 Chat Controller**

Create `api/src/Puupees.HttpApi/Controllers/Ai/AiChatController.cs`:

```csharp
using System.Threading.Tasks;
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Mvc;
using Puupees.Ai;
using Puupees.Ai.Dtos;

namespace Puupees.Controllers.Ai;

[Authorize]
[ApiController]
[Route("api/ai/chat")]
public class AiChatController : PuupeesController
{
    private readonly IAiChatAppService _chatAppService;

    public AiChatController(IAiChatAppService chatAppService)
    {
        _chatAppService = chatAppService;
    }

    [HttpPost("completions")]
    public Task<AiChatCompletionDto> CreateCompletionAsync([FromBody] CreateAiChatCompletionDto input)
    {
        return _chatAppService.CreateCompletionAsync(input);
    }
}
```

- [ ] **Step 2: 创建 Images Controller**

Create `api/src/Puupees.HttpApi/Controllers/Ai/AiImagesController.cs`:

```csharp
using System.Threading.Tasks;
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Mvc;
using Puupees.Ai;
using Puupees.Ai.Dtos;

namespace Puupees.Controllers.Ai;

[Authorize]
[ApiController]
[Route("api/ai/images")]
public class AiImagesController : PuupeesController
{
    private readonly IAiImageAppService _imageAppService;

    public AiImagesController(IAiImageAppService imageAppService)
    {
        _imageAppService = imageAppService;
    }

    [HttpPost("generations")]
    public Task<AiJobDto> CreateGenerationAsync([FromBody] CreateAiImageGenerationDto input)
    {
        return _imageAppService.CreateGenerationAsync(input);
    }

    [HttpPost("edits")]
    public Task<AiJobDto> CreateEditAsync([FromBody] CreateAiImageEditDto input)
    {
        return _imageAppService.CreateEditAsync(input);
    }
}
```

- [ ] **Step 3: 创建 Videos Controller**

Create `api/src/Puupees.HttpApi/Controllers/Ai/AiVideosController.cs`:

```csharp
using System.Threading.Tasks;
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Mvc;
using Puupees.Ai;
using Puupees.Ai.Dtos;

namespace Puupees.Controllers.Ai;

[Authorize]
[ApiController]
[Route("api/ai/videos")]
public class AiVideosController : PuupeesController
{
    private readonly IAiVideoAppService _videoAppService;

    public AiVideosController(IAiVideoAppService videoAppService)
    {
        _videoAppService = videoAppService;
    }

    [HttpPost("generations")]
    public Task<AiJobDto> CreateGenerationAsync([FromBody] CreateAiVideoGenerationDto input)
    {
        return _videoAppService.CreateGenerationAsync(input);
    }

    [HttpPost("edits")]
    public Task<AiJobDto> CreateEditAsync([FromBody] CreateAiVideoEditDto input)
    {
        return _videoAppService.CreateEditAsync(input);
    }
}
```

- [ ] **Step 4: 创建 Jobs Controller**

Create `api/src/Puupees.HttpApi/Controllers/Ai/AiJobsController.cs`:

```csharp
using System.Threading.Tasks;
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Mvc;
using Puupees.Ai;
using Puupees.Ai.Dtos;

namespace Puupees.Controllers.Ai;

[Authorize]
[ApiController]
[Route("api/ai/jobs")]
public class AiJobsController : PuupeesController
{
    private readonly IAiJobAppService _jobAppService;

    public AiJobsController(IAiJobAppService jobAppService)
    {
        _jobAppService = jobAppService;
    }

    [HttpGet("{id}")]
    public Task<AiJobDto> GetAsync(string id)
    {
        return _jobAppService.GetAsync(id);
    }

    [HttpPost("{id}/cancel")]
    public Task<AiJobDto> CancelAsync(string id)
    {
        return _jobAppService.CancelAsync(id);
    }
}
```

- [ ] **Step 5: 构建 HttpApi 项目**

Run:

```bash
cd api
dotnet build src/Puupees.HttpApi/Puupees.HttpApi.csproj
```

Expected: `Build succeeded.`，Controller 路由和注入类型编译通过。

- [ ] **Step 6: 提交 Puupee 原生 Controller**

```bash
git add api/src/Puupees.HttpApi/Controllers/Ai/AiChatController.cs api/src/Puupees.HttpApi/Controllers/Ai/AiImagesController.cs api/src/Puupees.HttpApi/Controllers/Ai/AiVideosController.cs api/src/Puupees.HttpApi/Controllers/Ai/AiJobsController.cs
git commit -m "feat(api): 添加 AI Gateway 原生路由"
```

## Task 7: 添加 OpenAI 兼容 Controller

**Files:**
- Create: `api/src/Puupees.HttpApi/Controllers/Ai/OpenAiCompatibleChatController.cs`
- Create: `api/src/Puupees.HttpApi/Controllers/Ai/OpenAiCompatibleImagesController.cs`
- Create: `api/src/Puupees.HttpApi/Controllers/Ai/OpenAiCompatibleVideosController.cs`

- [ ] **Step 1: 创建 OpenAI Chat Controller**

Create `api/src/Puupees.HttpApi/Controllers/Ai/OpenAiCompatibleChatController.cs`:

```csharp
using System.Threading.Tasks;
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Mvc;
using Puupees.Ai;
using Puupees.Ai.Dtos;

namespace Puupees.Controllers.Ai;

[Authorize]
[ApiController]
[Route("api/ai/v1/chat/completions")]
public class OpenAiCompatibleChatController : PuupeesController
{
    private readonly IAiChatAppService _chatAppService;

    public OpenAiCompatibleChatController(IAiChatAppService chatAppService)
    {
        _chatAppService = chatAppService;
    }

    [HttpPost]
    public async Task<AiChatCompletionDto> CreateAsync([FromBody] OpenAiChatCompletionRequestDto input)
    {
        return await _chatAppService.CreateCompletionAsync(input.ToPuupeeInput());
    }
}
```

- [ ] **Step 2: 创建 OpenAI Images Controller**

Create `api/src/Puupees.HttpApi/Controllers/Ai/OpenAiCompatibleImagesController.cs`:

```csharp
using System.Collections.Generic;
using System.Threading.Tasks;
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Http;
using Microsoft.AspNetCore.Mvc;
using Puupees.Ai;
using Puupees.Ai.Dtos;

namespace Puupees.Controllers.Ai;

[Authorize]
[ApiController]
[Route("api/ai/v1/images")]
public class OpenAiCompatibleImagesController : PuupeesController
{
    private readonly IAiImageAppService _imageAppService;

    public OpenAiCompatibleImagesController(IAiImageAppService imageAppService)
    {
        _imageAppService = imageAppService;
    }

    [HttpPost("generations")]
    public async Task<OpenAiImageResponseDto> CreateGenerationAsync([FromBody] OpenAiImageGenerationRequestDto input)
    {
        var job = await _imageAppService.CreateGenerationAsync(input.ToPuupeeInput());
        return job.ToOpenAiImageResponse();
    }

    [HttpPost("edits")]
    [Consumes("multipart/form-data")]
    public async Task<OpenAiImageResponseDto> CreateEditAsync([FromForm] OpenAiImageEditForm input)
    {
        var editInput = new CreateAiImageEditDto
        {
            Model = input.Model,
            Prompt = input.Prompt,
            Size = input.Size,
            N = input.N
        };

        foreach (var image in input.Image)
        {
            editInput.InputImages.Add(new AiMediaReferenceDto
            {
                Key = image.FileName,
                MimeType = image.ContentType
            });
        }

        if (input.Mask != null)
        {
            editInput.Mask = new AiMediaReferenceDto
            {
                Key = input.Mask.FileName,
                MimeType = input.Mask.ContentType
            };
        }

        var job = await _imageAppService.CreateEditAsync(editInput);
        return job.ToOpenAiImageResponse();
    }

    public class OpenAiImageEditForm
    {
        public string Model { get; set; } = "puupee-image";
        public string Prompt { get; set; } = string.Empty;
        public List<IFormFile> Image { get; set; } = new();
        public IFormFile? Mask { get; set; }
        public string? Size { get; set; }
        public int N { get; set; } = 1;
    }
}
```

- [ ] **Step 3: 创建 OpenAI Videos Controller**

Create `api/src/Puupees.HttpApi/Controllers/Ai/OpenAiCompatibleVideosController.cs`:

```csharp
using System.Threading.Tasks;
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Mvc;
using Puupees.Ai;
using Puupees.Ai.Dtos;

namespace Puupees.Controllers.Ai;

[Authorize]
[ApiController]
[Route("api/ai/v1/videos")]
public class OpenAiCompatibleVideosController : PuupeesController
{
    private readonly IAiVideoAppService _videoAppService;
    private readonly IAiJobAppService _jobAppService;

    public OpenAiCompatibleVideosController(
        IAiVideoAppService videoAppService,
        IAiJobAppService jobAppService)
    {
        _videoAppService = videoAppService;
        _jobAppService = jobAppService;
    }

    [HttpPost]
    public async Task<OpenAiVideoResponseDto> CreateAsync([FromBody] OpenAiVideoRequestDto input)
    {
        AiJobDto job;

        if (input.InputImages.Count == 0)
        {
            job = await _videoAppService.CreateGenerationAsync(new CreateAiVideoGenerationDto
            {
                Model = input.Model,
                Prompt = input.Prompt,
                Width = input.Width,
                Height = input.Height,
                DurationSeconds = input.DurationSeconds,
                Metadata = input.Metadata
            });
        }
        else
        {
            job = await _videoAppService.CreateEditAsync(new CreateAiVideoEditDto
            {
                Model = input.Model,
                Prompt = input.Prompt,
                InputImages = input.InputImages,
                Width = input.Width,
                Height = input.Height,
                DurationSeconds = input.DurationSeconds,
                Metadata = input.Metadata
            });
        }

        return job.ToOpenAiVideoResponse();
    }

    [HttpGet("{id}")]
    public async Task<OpenAiVideoResponseDto> GetAsync(string id)
    {
        var job = await _jobAppService.GetAsync(id);
        return job.ToOpenAiVideoResponse();
    }
}
```

- [ ] **Step 4: 构建 HttpApi 项目**

Run:

```bash
cd api
dotnet build src/Puupees.HttpApi/Puupees.HttpApi.csproj
```

Expected: `Build succeeded.`，没有 `IFormFile`、`Consumes` 或映射扩展方法相关编译错误。

- [ ] **Step 5: 运行 AI 相关测试**

Run:

```bash
cd api
dotnet test test/Puupees.Application.Tests/Puupees.Application.Tests.csproj --filter FullyQualifiedName~Ai
```

Expected: PASS，AI Gateway AppService 测试和 OpenAI 映射测试全部通过。

- [ ] **Step 6: 提交 OpenAI 兼容 Controller**

```bash
git add api/src/Puupees.HttpApi/Controllers/Ai/OpenAiCompatibleChatController.cs api/src/Puupees.HttpApi/Controllers/Ai/OpenAiCompatibleImagesController.cs api/src/Puupees.HttpApi/Controllers/Ai/OpenAiCompatibleVideosController.cs
git commit -m "feat(api): 添加 OpenAI 兼容 AI 路由"
```

## Task 8: 添加 Controller 路由和授权反射测试

**Files:**
- Modify: `api/test/Puupees.Application.Tests/Puupees.Application.Tests.csproj`
- Create: `api/test/Puupees.Application.Tests/Ai/AiControllerRouteAndSecurityTests.cs`

- [ ] **Step 1: 让测试项目引用 HttpApi 项目**

Modify `api/test/Puupees.Application.Tests/Puupees.Application.Tests.csproj` and add the `Puupees.HttpApi` project reference inside the existing first `<ItemGroup>`:

```xml
<ProjectReference Include="..\..\src\Puupees.HttpApi\Puupees.HttpApi.csproj" />
```

After the edit, the first `<ItemGroup>` should be:

```xml
<ItemGroup>
  <ProjectReference Include="..\..\src\Puupees.Application\Puupees.Application.csproj" />
  <ProjectReference Include="..\..\src\Puupees.HttpApi\Puupees.HttpApi.csproj" />
  <ProjectReference Include="..\Puupees.Domain.Tests\Puupees.Domain.Tests.csproj" />
</ItemGroup>
```

- [ ] **Step 2: 创建 Controller 路由和授权测试**

Create `api/test/Puupees.Application.Tests/Ai/AiControllerRouteAndSecurityTests.cs`:

```csharp
using System;
using System.Reflection;
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Mvc;
using Puupees.Controllers.Ai;
using Shouldly;
using Xunit;

namespace Puupees.Tests.Ai;

public class AiControllerRouteAndSecurityTests
{
    [Theory]
    [InlineData(typeof(AiChatController), "api/ai/chat")]
    [InlineData(typeof(AiImagesController), "api/ai/images")]
    [InlineData(typeof(AiVideosController), "api/ai/videos")]
    [InlineData(typeof(AiJobsController), "api/ai/jobs")]
    [InlineData(typeof(OpenAiCompatibleChatController), "api/ai/v1/chat/completions")]
    [InlineData(typeof(OpenAiCompatibleImagesController), "api/ai/v1/images")]
    [InlineData(typeof(OpenAiCompatibleVideosController), "api/ai/v1/videos")]
    public void Ai_Controller_Should_Have_Fixed_Route_And_Authorize_Attribute(
        Type controllerType,
        string expectedRoute)
    {
        controllerType.GetCustomAttribute<ApiControllerAttribute>().ShouldNotBeNull();
        controllerType.GetCustomAttribute<AuthorizeAttribute>().ShouldNotBeNull();

        var route = controllerType.GetCustomAttribute<RouteAttribute>();
        route.ShouldNotBeNull();
        route!.Template.ShouldBe(expectedRoute);
    }
}
```

- [ ] **Step 3: 运行路由和授权测试**

Run:

```bash
cd api
dotnet test test/Puupees.Application.Tests/Puupees.Application.Tests.csproj --filter FullyQualifiedName~AiControllerRouteAndSecurityTests
```

Expected: PASS，7 个 Controller 均有固定路由、`[ApiController]` 和 `[Authorize]`。

- [ ] **Step 4: 提交 Controller 测试**

```bash
git add api/test/Puupees.Application.Tests/Puupees.Application.Tests.csproj api/test/Puupees.Application.Tests/Ai/AiControllerRouteAndSecurityTests.cs
git commit -m "test(api): 覆盖 AI Gateway 路由和授权"
```

## Task 9: 全量验证和收尾

**Files:**
- Verify only.

- [ ] **Step 1: 构建 API 关键项目**

Run:

```bash
cd api
dotnet build src/Puupees.Application.Contracts/Puupees.Application.Contracts.csproj
dotnet build src/Puupees.Application/Puupees.Application.csproj
dotnet build src/Puupees.HttpApi/Puupees.HttpApi.csproj
```

Expected: 三个命令均输出 `Build succeeded.`。

- [ ] **Step 2: 运行应用测试**

Run:

```bash
cd api
dotnet test test/Puupees.Application.Tests/Puupees.Application.Tests.csproj
```

Expected: PASS。若出现与本次 AI Gateway 无关的既有失败，记录失败测试名称、错误摘要和本次已通过的 `--filter FullyQualifiedName~Ai` 结果，不修改无关模块。

- [ ] **Step 3: 检查 git 状态**

Run:

```bash
git status --short
```

Expected: 输出为空。

- [ ] **Step 4: 汇总最终结果**

在最终回复中说明：

- 新增了 Puupee 原生 AI Gateway 接口。
- 新增了 OpenAI 兼容入口。
- 第一阶段使用 Mock Provider，不依赖真实供应商 API Key。
- 列出实际运行的 build/test 命令和结果。
- 如果有未解决的既有测试失败，明确说明失败不属于本次改动范围。
