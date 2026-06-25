# 通用 AI 视觉与结构化抽取 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 在 API 中提供可被任意子应用复用的 AI 服务商管理、图片分析和结构化抽取能力，并让 developer 管理服务商，让小汪记物用真实通用接口替换 mock 识别。

**Architecture:** 先扩展 `Puupees.Ai` 通用网关：服务商与模型配置落库，Provider Router 按能力路由，`AiVisionAppService` 负责 OCR/标签/商品/图片摘要，`AiExtractionAppService` 负责 JSON 结构化抽取。Flutter 侧通过生成的 `felorx_api_client` 调用通用接口，developer 管理 provider，小汪记物只新增一个客户端适配服务，不新增后端专属 inventory recognition 接口。

**Tech Stack:** .NET 8、ABP Framework 8.1、Entity Framework Core、ASP.NET Core MVC、System.Text.Json、HttpClientFactory、xUnit、Shouldly、NSubstitute、Dart 3.8、Flutter、Riverpod、go_router、shadcn_flutter、OpenAPI Generator。

---

## Scope Check

这个 spec 横跨 API、客户端生成、developer 管理端和小汪记物接入。为了让每一段可独立验收，本计划按可工作的阶段拆任务：

- Task 1-5 交付可复用 API 能力，未配置真实服务商时 mock 可通过测试。
- Task 6 生成 Dart API 客户端。
- Task 7 交付 developer AI 服务商管理入口。
- Task 8-9 让小汪记物调用通用 AI 能力并保留确认页。
- Task 10 做整体验证。

若需要拆 PR，以 Task 5、Task 7、Task 9 作为自然边界。

## File Structure

- Create: `api/src/Puupees.Domain.Shared/Ai/AiProviderType.cs`
- Create: `api/src/Puupees.Domain.Shared/Ai/AiCapability.cs`
- Create: `api/src/Puupees.Domain/Ai/AiProvider.cs`
- Create: `api/src/Puupees.Domain/Ai/AiModel.cs`
- Modify: `api/src/Puupees.EntityFrameworkCore/EntityFrameworkCore/PuupeesDbContext.cs`
- Generated: `api/src/Puupees.EntityFrameworkCore/Migrations/*_AddAiProvidersAndModels.cs`
- Generated: `api/src/Puupees.EntityFrameworkCore/Migrations/*_AddAiProvidersAndModels.Designer.cs`
- Modify: `api/src/Puupees.EntityFrameworkCore/Migrations/PuupeesDbContextModelSnapshot.cs`
- Create: `api/src/Puupees.Application.Contracts/Ai/Dtos/AiProviderDtos.cs`
- Create: `api/src/Puupees.Application.Contracts/Ai/Dtos/AiVisionDtos.cs`
- Create: `api/src/Puupees.Application.Contracts/Ai/Dtos/AiExtractionDtos.cs`
- Create: `api/src/Puupees.Application.Contracts/Ai/IAiProviderAppService.cs`
- Create: `api/src/Puupees.Application.Contracts/Ai/IAiVisionAppService.cs`
- Create: `api/src/Puupees.Application.Contracts/Ai/IAiExtractionAppService.cs`
- Modify: `api/src/Puupees.Application.Contracts/Ai/Dtos/AiEnums.cs`
- Modify: `api/src/Puupees.Application/PuupeesApplicationAutoMapperProfile.cs`
- Modify: `api/src/Puupees.Application/PuupeesApplicationModule.cs`
- Create: `api/src/Puupees.Application/Ai/AiProviderAppService.cs`
- Create: `api/src/Puupees.Application/Ai/AiVisionAppService.cs`
- Create: `api/src/Puupees.Application/Ai/AiExtractionAppService.cs`
- Create: `api/src/Puupees.Application/Ai/Providers/IAiVisionProvider.cs`
- Create: `api/src/Puupees.Application/Ai/Providers/IAiStructuredExtractionProvider.cs`
- Modify: `api/src/Puupees.Application/Ai/Providers/IAiProviderRouter.cs`
- Modify: `api/src/Puupees.Application/Ai/Providers/DefaultAiProviderRouter.cs`
- Modify: `api/src/Puupees.Application/Ai/Providers/MockAiProvider.cs`
- Create: `api/src/Puupees.Application/Ai/Providers/AiProviderRegistry.cs`
- Create: `api/src/Puupees.Application/Ai/Providers/OpenAiCompatibleChatProvider.cs`
- Create: `api/src/Puupees.Application/Ai/Providers/TencentCloudVisionProvider.cs`
- Create: `api/src/Puupees.Application/Ai/Providers/TencentCloudRequestSigner.cs`
- Create: `api/src/Puupees.HttpApi/Controllers/Ai/AiProvidersController.cs`
- Create: `api/src/Puupees.HttpApi/Controllers/Ai/AiVisionController.cs`
- Create: `api/src/Puupees.HttpApi/Controllers/Ai/AiExtractionController.cs`
- Modify: `api/test/Puupees.Application.Tests/Ai/AiGatewayAppServiceTests.cs`
- Modify: `api/test/Puupees.Application.Tests/Ai/AiControllerRouteAndSecurityTests.cs`
- Create: `api/test/Puupees.Application.Tests/Ai/AiProviderRouterTests.cs`
- Create: `api/test/Puupees.Application.Tests/Ai/AiVisionAndExtractionTests.cs`
- Create: `api/test/Puupees.Application.Tests/Ai/TencentCloudProviderTests.cs`
- Generated/Modify: `packages/api/felorx_api_client/lib/**`
- Generated/Modify: `packages/api/felorx_api_client/doc/**`
- Generated/Modify: `packages/api/felorx_api_client/test/**`
- Modify: `apps/developer/lib/development/home_page.dart`
- Modify: `apps/developer/lib/router.dart`
- Create: `apps/developer/lib/ai-providers/home_page.dart`
- Create: `apps/developer/lib/ai-providers/list_view.dart`
- Create: `apps/developer/lib/ai-providers/list_view_item.dart`
- Create: `apps/developer/lib/ai-providers/edit_page.dart`
- Create: `apps/developer/lib/ai-providers/provider.dart`
- Modify: `apps/inventory/lib/services/inventory_recognition_service.dart`
- Modify: `apps/inventory/lib/providers/inventory_providers.dart`
- Modify: `apps/inventory/lib/providers/inventory_providers.g.dart`
- Create: `apps/inventory/lib/services/inventory_image_upload_service.dart`
- Create: `apps/inventory/lib/services/api_inventory_recognition_service.dart`
- Modify: `apps/inventory/test/inventory_recognition_service_test.dart`
- Create: `apps/inventory/test/api_inventory_recognition_service_test.dart`

---

### Task 1: API Domain Model and EF Mapping

**Files:**
- Create: `api/src/Puupees.Domain.Shared/Ai/AiProviderType.cs`
- Create: `api/src/Puupees.Domain.Shared/Ai/AiCapability.cs`
- Create: `api/src/Puupees.Domain/Ai/AiProvider.cs`
- Create: `api/src/Puupees.Domain/Ai/AiModel.cs`
- Modify: `api/src/Puupees.EntityFrameworkCore/EntityFrameworkCore/PuupeesDbContext.cs`
- Generated: `api/src/Puupees.EntityFrameworkCore/Migrations/*_AddAiProvidersAndModels.cs`
- Generated: `api/src/Puupees.EntityFrameworkCore/Migrations/*_AddAiProvidersAndModels.Designer.cs`
- Modify: `api/src/Puupees.EntityFrameworkCore/Migrations/PuupeesDbContextModelSnapshot.cs`

- [ ] **Step 1: Write the failing DbContext model test**

Create `api/test/Puupees.Application.Tests/Ai/AiProviderModelTests.cs`:

```csharp
using System;
using System.Linq;
using Microsoft.EntityFrameworkCore;
using Puupees.Ai;
using Puupees.EntityFrameworkCore;
using Shouldly;
using Xunit;

namespace Puupees.Tests.Ai;

public class AiProviderModelTests
{
    [Fact]
    public void DbContext_Should_Expose_AiProviders_And_AiModels()
    {
        var options = new DbContextOptionsBuilder<PuupeesDbContext>()
            .UseInMemoryDatabase($"ai-provider-model-{Guid.NewGuid():N}")
            .Options;

        using var db = new PuupeesDbContext(options);
        var providerType = db.Model.FindEntityType(typeof(AiProvider));
        var modelType = db.Model.FindEntityType(typeof(AiModel));

        providerType.ShouldNotBeNull();
        modelType.ShouldNotBeNull();
        providerType!.GetTableName().ShouldBe("PuupeesAiProviders");
        modelType!.GetTableName().ShouldBe("PuupeesAiModels");
        modelType.GetForeignKeys().Any(fk => fk.PrincipalEntityType.ClrType == typeof(AiProvider)).ShouldBeTrue();
    }
}
```

- [ ] **Step 2: Run the test and verify RED**

Run:

```bash
cd api
dotnet test test/Puupees.Application.Tests/Puupees.Application.Tests.csproj --filter AiProviderModelTests
```

Expected: FAIL because `AiProvider` and `AiModel` do not exist.

- [ ] **Step 3: Add shared enums**

Create `api/src/Puupees.Domain.Shared/Ai/AiProviderType.cs`:

```csharp
using System.Text.Json.Serialization;

namespace Puupees.Ai;

[JsonConverter(typeof(JsonStringEnumConverter))]
public enum AiProviderType
{
    Mock,
    TencentCloud,
    OpenAiCompatible
}
```

Create `api/src/Puupees.Domain.Shared/Ai/AiCapability.cs`:

```csharp
using System.Text.Json.Serialization;

namespace Puupees.Ai;

[JsonConverter(typeof(JsonStringEnumConverter))]
public enum AiCapability
{
    Chat,
    Vision,
    Ocr,
    ImageLabel,
    ProductDetect,
    StructuredExtraction,
    Caption
}
```

- [ ] **Step 4: Add domain entities**

Create `api/src/Puupees.Domain/Ai/AiProvider.cs`:

```csharp
using System;
using System.Collections.Generic;
using Volo.Abp.Domain.Entities.Auditing;

namespace Puupees.Ai;

public class AiProvider : FullAuditedAggregateRoot<Guid>
{
    public string Name { get; set; } = string.Empty;
    public string DisplayName { get; set; } = string.Empty;
    public AiProviderType ProviderType { get; set; }
    public string? BaseUrl { get; set; }
    public string? Region { get; set; }
    public bool Enabled { get; set; } = true;
    public List<AiCapability> Capabilities { get; set; } = new();
    public string? EncryptedSecret { get; set; }
    public Dictionary<string, string> Metadata { get; set; } = new();
    public ICollection<AiModel> Models { get; set; } = new List<AiModel>();
}
```

Create `api/src/Puupees.Domain/Ai/AiModel.cs`:

```csharp
using System;
using System.Collections.Generic;
using Volo.Abp.Domain.Entities.Auditing;

namespace Puupees.Ai;

public class AiModel : FullAuditedEntity<Guid>
{
    public Guid ProviderId { get; set; }
    public AiProvider? Provider { get; set; }
    public string Name { get; set; } = string.Empty;
    public string DisplayName { get; set; } = string.Empty;
    public List<AiCapability> Capabilities { get; set; } = new();
    public bool Enabled { get; set; } = true;
    public bool IsDefault { get; set; }
    public Dictionary<string, string> DefaultParameters { get; set; } = new();
}
```

- [ ] **Step 5: Register DbSets and model conversions**

Modify `api/src/Puupees.EntityFrameworkCore/EntityFrameworkCore/PuupeesDbContext.cs`:

```csharp
public DbSet<AiProvider> AiProviders { get; set; }
public DbSet<AiModel> AiModels { get; set; }
```

Add enum conversion in `ConfigureEnumConversions`:

```csharp
builder.Properties<AiProviderType>()
    .HaveConversion<string>();
```

Add model configuration near the existing app/message entity blocks:

```csharp
builder.Entity<AiProvider>(b =>
{
    b.ToTable(PuupeesConsts.DbTablePrefix + "AiProviders", PuupeesConsts.DbSchema);
    b.ConfigureByConvention();
    b.HasIndex(it => it.Name).IsUnique();

    b.Property(it => it.Capabilities)
        .HasConversion(
            v => JsonSerializer.Serialize(v, jsonSerializerOptions),
            v => JsonSerializer.Deserialize<List<AiCapability>>(v, jsonSerializerOptions) ?? new List<AiCapability>())
        .Metadata.SetValueComparer(new ValueComparer<List<AiCapability>>(
            (c1, c2) => c1 != null && c2 != null && c1.SequenceEqual(c2),
            c => c.Aggregate(0, (a, v) => HashCode.Combine(a, v.GetHashCode())),
            c => c.ToList()));

    b.Property(it => it.Metadata)
        .HasConversion(
            v => JsonSerializer.Serialize(v, jsonSerializerOptions),
            v => JsonSerializer.Deserialize<Dictionary<string, string>>(v, jsonSerializerOptions) ?? new Dictionary<string, string>())
        .Metadata.SetValueComparer(new ValueComparer<Dictionary<string, string>>(
            (c1, c2) => c1 != null && c2 != null && c1.SequenceEqual(c2),
            c => c.Aggregate(0, (a, v) => HashCode.Combine(a, v.Key.GetHashCode(), v.Value.GetHashCode())),
            c => new Dictionary<string, string>(c)));

    b.HasMany(it => it.Models)
        .WithOne(it => it.Provider)
        .HasForeignKey(it => it.ProviderId)
        .OnDelete(DeleteBehavior.Cascade);
});

builder.Entity<AiModel>(b =>
{
    b.ToTable(PuupeesConsts.DbTablePrefix + "AiModels", PuupeesConsts.DbSchema);
    b.ConfigureByConvention();
    b.HasIndex(it => new { it.ProviderId, it.Name }).IsUnique();
    b.HasIndex(it => new { it.ProviderId, it.IsDefault });

    b.Property(it => it.Capabilities)
        .HasConversion(
            v => JsonSerializer.Serialize(v, jsonSerializerOptions),
            v => JsonSerializer.Deserialize<List<AiCapability>>(v, jsonSerializerOptions) ?? new List<AiCapability>())
        .Metadata.SetValueComparer(new ValueComparer<List<AiCapability>>(
            (c1, c2) => c1 != null && c2 != null && c1.SequenceEqual(c2),
            c => c.Aggregate(0, (a, v) => HashCode.Combine(a, v.GetHashCode())),
            c => c.ToList()));

    b.Property(it => it.DefaultParameters)
        .HasConversion(
            v => JsonSerializer.Serialize(v, jsonSerializerOptions),
            v => JsonSerializer.Deserialize<Dictionary<string, string>>(v, jsonSerializerOptions) ?? new Dictionary<string, string>())
        .Metadata.SetValueComparer(new ValueComparer<Dictionary<string, string>>(
            (c1, c2) => c1 != null && c2 != null && c1.SequenceEqual(c2),
            c => c.Aggregate(0, (a, v) => HashCode.Combine(a, v.Key.GetHashCode(), v.Value.GetHashCode())),
            c => new Dictionary<string, string>(c)));
});
```

- [ ] **Step 6: Add EF package for in-memory tests**

Modify `api/test/Puupees.Application.Tests/Puupees.Application.Tests.csproj`:

```xml
<PackageReference Include="Microsoft.EntityFrameworkCore.InMemory" Version="8.0.0" />
```

- [ ] **Step 7: Run model test and verify GREEN**

Run:

```bash
cd api
dotnet test test/Puupees.Application.Tests/Puupees.Application.Tests.csproj --filter AiProviderModelTests
```

Expected: PASS.

- [ ] **Step 8: Generate migration**

Run:

```bash
cd api
task db-add-migration -- AddAiProvidersAndModels
```

Expected: creates migration files under `api/src/Puupees.EntityFrameworkCore/Migrations/` and updates `PuupeesDbContextModelSnapshot.cs`.

- [ ] **Step 9: Commit**

```bash
git add api/src/Puupees.Domain.Shared/Ai api/src/Puupees.Domain/Ai api/src/Puupees.EntityFrameworkCore api/test/Puupees.Application.Tests
git commit -m "feat(api): 添加 AI 服务商模型"
```

---

### Task 2: Provider Management Contracts and AppService

**Files:**
- Create: `api/src/Puupees.Application.Contracts/Ai/Dtos/AiProviderDtos.cs`
- Create: `api/src/Puupees.Application.Contracts/Ai/IAiProviderAppService.cs`
- Modify: `api/src/Puupees.Application/PuupeesApplicationAutoMapperProfile.cs`
- Create: `api/src/Puupees.Application/Ai/AiProviderAppService.cs`
- Create: `api/src/Puupees.HttpApi/Controllers/Ai/AiProvidersController.cs`
- Modify: `api/test/Puupees.Application.Tests/Ai/AiControllerRouteAndSecurityTests.cs`

- [ ] **Step 1: Write failing controller security test**

Modify `api/test/Puupees.Application.Tests/Ai/AiControllerRouteAndSecurityTests.cs` and add inline data:

```csharp
[InlineData(typeof(AiProvidersController), "api/ai/providers")]
```

Run:

```bash
cd api
dotnet test test/Puupees.Application.Tests/Puupees.Application.Tests.csproj --filter AiControllerRouteAndSecurityTests
```

Expected: FAIL because `AiProvidersController` does not exist.

- [ ] **Step 2: Add provider DTOs**

Create `api/src/Puupees.Application.Contracts/Ai/Dtos/AiProviderDtos.cs`:

```csharp
using System;
using System.Collections.Generic;
using System.Text.Json.Serialization;
using Volo.Abp.Application.Dtos;

namespace Puupees.Ai.Dtos;

[Serializable]
public class AiModelDto : FullAuditedEntityDto<Guid>
{
    [JsonPropertyName("provider_id")]
    public Guid ProviderId { get; set; }

    [JsonPropertyName("name")]
    public string Name { get; set; } = string.Empty;

    [JsonPropertyName("display_name")]
    public string DisplayName { get; set; } = string.Empty;

    [JsonPropertyName("capabilities")]
    public List<AiCapability> Capabilities { get; set; } = new();

    [JsonPropertyName("enabled")]
    public bool Enabled { get; set; }

    [JsonPropertyName("is_default")]
    public bool IsDefault { get; set; }

    [JsonPropertyName("default_parameters")]
    public Dictionary<string, string> DefaultParameters { get; set; } = new();
}

[Serializable]
public class AiProviderDto : FullAuditedEntityDto<Guid>
{
    [JsonPropertyName("name")]
    public string Name { get; set; } = string.Empty;

    [JsonPropertyName("display_name")]
    public string DisplayName { get; set; } = string.Empty;

    [JsonPropertyName("provider_type")]
    public AiProviderType ProviderType { get; set; }

    [JsonPropertyName("base_url")]
    public string? BaseUrl { get; set; }

    [JsonPropertyName("region")]
    public string? Region { get; set; }

    [JsonPropertyName("enabled")]
    public bool Enabled { get; set; }

    [JsonPropertyName("capabilities")]
    public List<AiCapability> Capabilities { get; set; } = new();

    [JsonPropertyName("secret_configured")]
    public bool SecretConfigured { get; set; }

    [JsonPropertyName("metadata")]
    public Dictionary<string, string> Metadata { get; set; } = new();

    [JsonPropertyName("models")]
    public List<AiModelDto> Models { get; set; } = new();
}

[Serializable]
public class CreateOrUpdateAiModelDto
{
    public string Name { get; set; } = string.Empty;
    public string DisplayName { get; set; } = string.Empty;
    public List<AiCapability> Capabilities { get; set; } = new();
    public bool Enabled { get; set; } = true;
    public bool IsDefault { get; set; }
    public Dictionary<string, string> DefaultParameters { get; set; } = new();
}

[Serializable]
public class CreateOrUpdateAiProviderDto
{
    public string Name { get; set; } = string.Empty;
    public string DisplayName { get; set; } = string.Empty;
    public AiProviderType ProviderType { get; set; }
    public string? BaseUrl { get; set; }
    public string? Region { get; set; }
    public bool Enabled { get; set; } = true;
    public List<AiCapability> Capabilities { get; set; } = new();
    public string? Secret { get; set; }
    public bool ClearSecret { get; set; }
    public Dictionary<string, string> Metadata { get; set; } = new();
    public List<CreateOrUpdateAiModelDto> Models { get; set; } = new();
}

[Serializable]
public class AiProviderGetListInput : PagedAndSortedResultRequestDto
{
    public string? Filter { get; set; }
    public AiProviderType? ProviderType { get; set; }
    public AiCapability? Capability { get; set; }
    public bool? Enabled { get; set; }
}

[Serializable]
public class SetAiProviderEnabledDto
{
    public bool Enabled { get; set; }
}

[Serializable]
public class SetDefaultAiModelDto
{
    public Guid ModelId { get; set; }
    public AiCapability Capability { get; set; }
}

[Serializable]
public class TestAiProviderDto
{
    public AiCapability Capability { get; set; }
    public string? Prompt { get; set; }
    public string? ImageUrl { get; set; }
}
```

- [ ] **Step 3: Add provider AppService contract**

Create `api/src/Puupees.Application.Contracts/Ai/IAiProviderAppService.cs`:

```csharp
using System;
using System.Threading.Tasks;
using Puupees.Ai.Dtos;
using Volo.Abp.Application.Dtos;
using Volo.Abp.Application.Services;

namespace Puupees.Ai;

public interface IAiProviderAppService : IApplicationService
{
    Task<AiProviderDto> CreateAsync(CreateOrUpdateAiProviderDto input);
    Task<AiProviderDto> UpdateAsync(Guid id, CreateOrUpdateAiProviderDto input);
    Task DeleteAsync(Guid id);
    Task<AiProviderDto> GetAsync(Guid id);
    Task<PagedResultDto<AiProviderDto>> GetListAsync(AiProviderGetListInput input);
    Task<AiProviderDto> SetEnabledAsync(Guid id, SetAiProviderEnabledDto input);
    Task<AiProviderDto> SetDefaultModelAsync(SetDefaultAiModelDto input);
    Task<AiProviderDto> TestAsync(Guid id, TestAiProviderDto input);
}
```

- [ ] **Step 4: Add AutoMapper mappings**

Modify `api/src/Puupees.Application/PuupeesApplicationAutoMapperProfile.cs`:

```csharp
CreateMap<AiProvider, AiProviderDto>()
    .ForMember(d => d.SecretConfigured, o => o.MapFrom(s => !string.IsNullOrEmpty(s.EncryptedSecret)));
CreateMap<AiModel, AiModelDto>();
CreateMap<CreateOrUpdateAiProviderDto, AiProvider>(MemberList.Source)
    .ForSourceMember(s => s.Secret, o => o.DoNotValidate())
    .ForSourceMember(s => s.ClearSecret, o => o.DoNotValidate())
    .ForMember(d => d.EncryptedSecret, o => o.Ignore())
    .ForMember(d => d.Models, o => o.Ignore());
CreateMap<CreateOrUpdateAiModelDto, AiModel>(MemberList.Source);
```

- [ ] **Step 5: Implement AppService**

Create `api/src/Puupees.Application/Ai/AiProviderAppService.cs`:

```csharp
using System;
using System.Collections.Generic;
using System.Linq;
using System.Threading.Tasks;
using Microsoft.AspNetCore.Authorization;
using Microsoft.EntityFrameworkCore;
using Puupees.Ai.Dtos;
using Volo.Abp;
using Volo.Abp.Application.Dtos;
using Volo.Abp.Domain.Repositories;
using Volo.Abp.Linq;

namespace Puupees.Ai;

[Authorize]
public class AiProviderAppService : PuupeesAppServiceBase, IAiProviderAppService
{
    private readonly IRepository<AiProvider, Guid> _providerRepo;
    private readonly IRepository<AiModel, Guid> _modelRepo;

    public AiProviderAppService(
        IRepository<AiProvider, Guid> providerRepo,
        IRepository<AiModel, Guid> modelRepo)
    {
        _providerRepo = providerRepo;
        _modelRepo = modelRepo;
    }

    public async Task<AiProviderDto> CreateAsync(CreateOrUpdateAiProviderDto input)
    {
        await EnsureProviderNameAvailableAsync(input.Name, null);
        var provider = ObjectMapper.Map<CreateOrUpdateAiProviderDto, AiProvider>(input);
        provider.EncryptedSecret = ProtectSecret(input.Secret);
        provider = await _providerRepo.InsertAsync(provider, autoSave: true);
        await ReplaceModelsAsync(provider.Id, input.Models);
        return await GetAsync(provider.Id);
    }

    public async Task<AiProviderDto> UpdateAsync(Guid id, CreateOrUpdateAiProviderDto input)
    {
        await EnsureProviderNameAvailableAsync(input.Name, id);
        var provider = await _providerRepo.GetAsync(id);
        ObjectMapper.Map(input, provider);
        if (input.ClearSecret)
        {
            provider.EncryptedSecret = null;
        }
        else if (!string.IsNullOrWhiteSpace(input.Secret))
        {
            provider.EncryptedSecret = ProtectSecret(input.Secret);
        }

        await _providerRepo.UpdateAsync(provider, autoSave: true);
        await ReplaceModelsAsync(provider.Id, input.Models);
        return await GetAsync(provider.Id);
    }

    public Task DeleteAsync(Guid id)
    {
        return _providerRepo.DeleteAsync(id);
    }

    public async Task<AiProviderDto> GetAsync(Guid id)
    {
        var query = await _providerRepo.WithDetailsAsync(p => p.Models);
        var provider = await query.FirstAsync(p => p.Id == id);
        return ObjectMapper.Map<AiProvider, AiProviderDto>(provider);
    }

    public async Task<PagedResultDto<AiProviderDto>> GetListAsync(AiProviderGetListInput input)
    {
        var query = await _providerRepo.WithDetailsAsync(p => p.Models);
        if (!string.IsNullOrWhiteSpace(input.Filter))
        {
            var filter = input.Filter.Trim();
            query = query.Where(p => p.Name.Contains(filter) || p.DisplayName.Contains(filter));
        }
        if (input.ProviderType.HasValue)
        {
            query = query.Where(p => p.ProviderType == input.ProviderType.Value);
        }
        if (input.Capability.HasValue)
        {
            query = query.Where(p => p.Capabilities.Contains(input.Capability.Value));
        }
        if (input.Enabled.HasValue)
        {
            query = query.Where(p => p.Enabled == input.Enabled.Value);
        }

        var count = await query.CountAsync();
        var items = await query
            .OrderBy(p => p.Name)
            .PageBy(input)
            .ToListAsync();
        return new PagedResultDto<AiProviderDto>(
            count,
            ObjectMapper.Map<List<AiProvider>, List<AiProviderDto>>(items));
    }

    public async Task<AiProviderDto> SetEnabledAsync(Guid id, SetAiProviderEnabledDto input)
    {
        var provider = await _providerRepo.GetAsync(id);
        provider.Enabled = input.Enabled;
        await _providerRepo.UpdateAsync(provider, autoSave: true);
        return await GetAsync(id);
    }

    public async Task<AiProviderDto> SetDefaultModelAsync(SetDefaultAiModelDto input)
    {
        var model = await _modelRepo.GetAsync(input.ModelId);
        if (!model.Capabilities.Contains(input.Capability))
        {
            throw new BusinessException("Puupees.Ai:ModelCapabilityMismatch");
        }

        var query = await _modelRepo.GetQueryableAsync();
        var siblings = await query
            .Where(m => m.ProviderId == model.ProviderId && m.Capabilities.Contains(input.Capability))
            .ToListAsync();
        foreach (var sibling in siblings)
        {
            sibling.IsDefault = sibling.Id == model.Id;
        }
        await _modelRepo.UpdateManyAsync(siblings, autoSave: true);
        return await GetAsync(model.ProviderId);
    }

    public Task<AiProviderDto> TestAsync(Guid id, TestAiProviderDto input)
    {
        return GetAsync(id);
    }

    private async Task EnsureProviderNameAvailableAsync(string name, Guid? currentId)
    {
        var normalized = name.Trim();
        if (normalized.Length == 0)
        {
            throw new BusinessException("Puupees.Ai:ProviderNameRequired");
        }

        var query = await _providerRepo.GetQueryableAsync();
        var exists = await query.AnyAsync(p => p.Name == normalized && (!currentId.HasValue || p.Id != currentId.Value));
        if (exists)
        {
            throw new BusinessException("Puupees.Ai:ProviderNameExists");
        }
    }

    private async Task ReplaceModelsAsync(Guid providerId, List<CreateOrUpdateAiModelDto> inputs)
    {
        var query = await _modelRepo.GetQueryableAsync();
        var existing = await query.Where(m => m.ProviderId == providerId).ToListAsync();
        await _modelRepo.DeleteManyAsync(existing, autoSave: true);
        foreach (var input in inputs)
        {
            var model = ObjectMapper.Map<CreateOrUpdateAiModelDto, AiModel>(input);
            model.ProviderId = providerId;
            await _modelRepo.InsertAsync(model, autoSave: true);
        }
    }

    private static string? ProtectSecret(string? secret)
    {
        return string.IsNullOrWhiteSpace(secret)
            ? null
            : Convert.ToBase64String(System.Text.Encoding.UTF8.GetBytes(secret));
    }
}
```

- [ ] **Step 6: Add controller**

Create `api/src/Puupees.HttpApi/Controllers/Ai/AiProvidersController.cs`:

```csharp
using System;
using System.Threading.Tasks;
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Mvc;
using Puupees.Ai;
using Puupees.Ai.Dtos;
using Volo.Abp.Application.Dtos;

namespace Puupees.Controllers.Ai;

[Authorize]
[ApiController]
[Route("api/ai/providers")]
public class AiProvidersController : PuupeesController
{
    private readonly IAiProviderAppService _providerAppService;

    public AiProvidersController(IAiProviderAppService providerAppService)
    {
        _providerAppService = providerAppService;
    }

    [HttpGet]
    public Task<PagedResultDto<AiProviderDto>> GetListAsync([FromQuery] AiProviderGetListInput input)
    {
        return _providerAppService.GetListAsync(input);
    }

    [HttpGet("{id}")]
    public Task<AiProviderDto> GetAsync(Guid id)
    {
        return _providerAppService.GetAsync(id);
    }

    [HttpPost]
    public Task<AiProviderDto> CreateAsync([FromBody] CreateOrUpdateAiProviderDto input)
    {
        return _providerAppService.CreateAsync(input);
    }

    [HttpPut("{id}")]
    public Task<AiProviderDto> UpdateAsync(Guid id, [FromBody] CreateOrUpdateAiProviderDto input)
    {
        return _providerAppService.UpdateAsync(id, input);
    }

    [HttpDelete("{id}")]
    public Task DeleteAsync(Guid id)
    {
        return _providerAppService.DeleteAsync(id);
    }

    [HttpPost("{id}/enabled")]
    public Task<AiProviderDto> SetEnabledAsync(Guid id, [FromBody] SetAiProviderEnabledDto input)
    {
        return _providerAppService.SetEnabledAsync(id, input);
    }

    [HttpPost("default-model")]
    public Task<AiProviderDto> SetDefaultModelAsync([FromBody] SetDefaultAiModelDto input)
    {
        return _providerAppService.SetDefaultModelAsync(input);
    }

    [HttpPost("{id}/test")]
    public Task<AiProviderDto> TestAsync(Guid id, [FromBody] TestAiProviderDto input)
    {
        return _providerAppService.TestAsync(id, input);
    }
}
```

- [ ] **Step 7: Run route test and verify GREEN**

Run:

```bash
cd api
dotnet test test/Puupees.Application.Tests/Puupees.Application.Tests.csproj --filter AiControllerRouteAndSecurityTests
```

Expected: PASS.

- [ ] **Step 8: Commit**

```bash
git add api/src/Puupees.Application.Contracts/Ai api/src/Puupees.Application/Ai api/src/Puupees.Application/PuupeesApplicationAutoMapperProfile.cs api/src/Puupees.HttpApi/Controllers/Ai api/test/Puupees.Application.Tests/Ai
git commit -m "feat(api): 添加 AI 服务商管理接口"
```

---

### Task 3: Vision and Structured Extraction Contracts with Mock Provider

**Files:**
- Modify: `api/src/Puupees.Application.Contracts/Ai/Dtos/AiEnums.cs`
- Create: `api/src/Puupees.Application.Contracts/Ai/Dtos/AiVisionDtos.cs`
- Create: `api/src/Puupees.Application.Contracts/Ai/Dtos/AiExtractionDtos.cs`
- Create: `api/src/Puupees.Application.Contracts/Ai/IAiVisionAppService.cs`
- Create: `api/src/Puupees.Application.Contracts/Ai/IAiExtractionAppService.cs`
- Create: `api/src/Puupees.Application/Ai/AiVisionAppService.cs`
- Create: `api/src/Puupees.Application/Ai/AiExtractionAppService.cs`
- Create: `api/src/Puupees.Application/Ai/Providers/IAiVisionProvider.cs`
- Create: `api/src/Puupees.Application/Ai/Providers/IAiStructuredExtractionProvider.cs`
- Modify: `api/src/Puupees.Application/Ai/Providers/MockAiProvider.cs`
- Modify: `api/src/Puupees.Application/Ai/Providers/IAiProviderRouter.cs`
- Modify: `api/src/Puupees.Application/Ai/Providers/DefaultAiProviderRouter.cs`
- Create: `api/src/Puupees.HttpApi/Controllers/Ai/AiVisionController.cs`
- Create: `api/src/Puupees.HttpApi/Controllers/Ai/AiExtractionController.cs`
- Create: `api/test/Puupees.Application.Tests/Ai/AiVisionAndExtractionTests.cs`
- Modify: `api/test/Puupees.Application.Tests/Ai/AiControllerRouteAndSecurityTests.cs`

- [ ] **Step 1: Write failing service tests**

Create `api/test/Puupees.Application.Tests/Ai/AiVisionAndExtractionTests.cs`:

```csharp
using System.Collections.Generic;
using System.Text.Json;
using System.Threading.Tasks;
using Puupees.Ai;
using Puupees.Ai.Dtos;
using Puupees.Ai.Providers;
using Shouldly;
using Xunit;

namespace Puupees.Tests.Ai;

public class AiVisionAndExtractionTests
{
    private readonly IAiVisionAppService _visionAppService;
    private readonly IAiExtractionAppService _extractionAppService;

    public AiVisionAndExtractionTests()
    {
        var provider = new MockAiProvider();
        var router = new DefaultAiProviderRouter(provider);
        _visionAppService = new AiVisionAppService(router);
        _extractionAppService = new AiExtractionAppService(router);
    }

    [Fact]
    public async Task Should_Analyze_Image_With_Mock_Ocr_Labels_And_Products()
    {
        var result = await _visionAppService.AnalyzeImageAsync(new CreateAiImageAnalysisDto
        {
            Image = new AiMediaReferenceDto { Url = "https://cdn.puupee.com/test/desk.jpg", MimeType = "image/jpeg" },
            Tasks = { AiImageAnalysisTask.Ocr, AiImageAnalysisTask.Labels, AiImageAnalysisTask.Products, AiImageAnalysisTask.Caption },
            Locale = "zh-CN"
        });

        result.Provider.ShouldBe("mock");
        result.OcrTexts.ShouldContain(t => t.Text.Contains("iPhone"));
        result.Labels.ShouldContain(l => l.Name == "手机");
        result.Products.ShouldContain(p => p.Name.Contains("iPhone"));
        result.Caption.ShouldContain("桌面");
    }

    [Fact]
    public async Task Should_Create_Structured_Extraction_Json()
    {
        var result = await _extractionAppService.CreateStructuredExtractionAsync(new CreateAiStructuredExtractionDto
        {
            SchemaName = "inventory_asset_draft",
            JsonSchema = """{"type":"object","properties":{"name":{"type":"string"}},"required":["name"]}""",
            SourceText = "图片中包含 iPhone 15 Pro 和 Apple 充电器",
            Instruction = "提取资产草稿 JSON"
        });

        result.Provider.ShouldBe("mock");
        result.Json.RootElement.ValueKind.ShouldBe(JsonValueKind.Object);
        result.Json.RootElement.GetProperty("items").GetArrayLength().ShouldBeGreaterThan(0);
        result.Confidence.ShouldBeGreaterThan(0);
    }
}
```

- [ ] **Step 2: Run tests and verify RED**

Run:

```bash
cd api
dotnet test test/Puupees.Application.Tests/Puupees.Application.Tests.csproj --filter AiVisionAndExtractionTests
```

Expected: FAIL because DTOs and services do not exist.

- [ ] **Step 3: Add image analysis enum**

Modify `api/src/Puupees.Application.Contracts/Ai/Dtos/AiEnums.cs`:

```csharp
[JsonConverter(typeof(JsonStringEnumConverter))]
public enum AiImageAnalysisTask
{
    Ocr,
    Labels,
    Products,
    Caption
}
```

- [ ] **Step 4: Add vision DTOs**

Create `api/src/Puupees.Application.Contracts/Ai/Dtos/AiVisionDtos.cs`:

```csharp
using System;
using System.Collections.Generic;
using System.Text.Json.Serialization;

namespace Puupees.Ai.Dtos;

[Serializable]
public class CreateAiImageAnalysisDto
{
    [JsonPropertyName("image")]
    public AiMediaReferenceDto Image { get; set; } = new();

    [JsonPropertyName("tasks")]
    public List<AiImageAnalysisTask> Tasks { get; set; } = new();

    [JsonPropertyName("locale")]
    public string Locale { get; set; } = "zh-CN";

    [JsonPropertyName("provider")]
    public string? Provider { get; set; }

    [JsonPropertyName("model")]
    public string? Model { get; set; }

    [JsonPropertyName("metadata")]
    public Dictionary<string, string> Metadata { get; set; } = new();
}

[Serializable]
public class AiOcrTextDto
{
    public string Text { get; set; } = string.Empty;
    public double Confidence { get; set; }
    public List<int> Polygon { get; set; } = new();
}

[Serializable]
public class AiImageLabelDto
{
    public string Name { get; set; } = string.Empty;
    public string? Category { get; set; }
    public double Confidence { get; set; }
}

[Serializable]
public class AiImageProductDto
{
    public string Name { get; set; } = string.Empty;
    public string? Brand { get; set; }
    public string? Category { get; set; }
    public double Confidence { get; set; }
}

[Serializable]
public class AiProviderResultDto
{
    public string Provider { get; set; } = string.Empty;
    public string? Model { get; set; }
    public string? RequestId { get; set; }
    public long ElapsedMilliseconds { get; set; }
}

[Serializable]
public class AiImageAnalysisDto
{
    public string Provider { get; set; } = string.Empty;
    public string? Model { get; set; }
    public List<AiOcrTextDto> OcrTexts { get; set; } = new();
    public List<AiImageLabelDto> Labels { get; set; } = new();
    public List<AiImageProductDto> Products { get; set; } = new();
    public string? Caption { get; set; }
    public List<AiProviderResultDto> ProviderResults { get; set; } = new();
    public AiUsageDto? Usage { get; set; }
}
```

- [ ] **Step 5: Add extraction DTOs**

Create `api/src/Puupees.Application.Contracts/Ai/Dtos/AiExtractionDtos.cs`:

```csharp
using System;
using System.Collections.Generic;
using System.Text.Json;

namespace Puupees.Ai.Dtos;

[Serializable]
public class CreateAiStructuredExtractionDto
{
    public string SchemaName { get; set; } = string.Empty;
    public string JsonSchema { get; set; } = string.Empty;
    public string? SourceText { get; set; }
    public AiImageAnalysisDto? SourceVisionResult { get; set; }
    public string Instruction { get; set; } = string.Empty;
    public string? Provider { get; set; }
    public string? Model { get; set; }
    public Dictionary<string, string> Metadata { get; set; } = new();
}

[Serializable]
public class AiStructuredExtractionDto
{
    public JsonDocument Json { get; set; } = JsonDocument.Parse("{}");
    public double Confidence { get; set; }
    public List<string> MissingFields { get; set; } = new();
    public string? RawText { get; set; }
    public string Provider { get; set; } = string.Empty;
    public string? Model { get; set; }
    public AiUsageDto? Usage { get; set; }
}
```

- [ ] **Step 6: Add service contracts and provider interfaces**

Create `api/src/Puupees.Application.Contracts/Ai/IAiVisionAppService.cs`:

```csharp
using System.Threading.Tasks;
using Puupees.Ai.Dtos;
using Volo.Abp.Application.Services;

namespace Puupees.Ai;

public interface IAiVisionAppService : IApplicationService
{
    Task<AiImageAnalysisDto> AnalyzeImageAsync(CreateAiImageAnalysisDto input);
}
```

Create `api/src/Puupees.Application.Contracts/Ai/IAiExtractionAppService.cs`:

```csharp
using System.Threading.Tasks;
using Puupees.Ai.Dtos;
using Volo.Abp.Application.Services;

namespace Puupees.Ai;

public interface IAiExtractionAppService : IApplicationService
{
    Task<AiStructuredExtractionDto> CreateStructuredExtractionAsync(CreateAiStructuredExtractionDto input);
}
```

Create `api/src/Puupees.Application/Ai/Providers/IAiVisionProvider.cs`:

```csharp
using System.Threading.Tasks;
using Puupees.Ai.Dtos;

namespace Puupees.Ai.Providers;

public interface IAiVisionProvider
{
    string Name { get; }
    Task<AiImageAnalysisDto> AnalyzeImageAsync(CreateAiImageAnalysisDto input);
}
```

Create `api/src/Puupees.Application/Ai/Providers/IAiStructuredExtractionProvider.cs`:

```csharp
using System.Threading.Tasks;
using Puupees.Ai.Dtos;

namespace Puupees.Ai.Providers;

public interface IAiStructuredExtractionProvider
{
    string Name { get; }
    Task<AiStructuredExtractionDto> CreateStructuredExtractionAsync(CreateAiStructuredExtractionDto input);
}
```

- [ ] **Step 7: Extend router interface and implementation**

Modify `api/src/Puupees.Application/Ai/Providers/IAiProviderRouter.cs`:

```csharp
Task<IAiVisionProvider> GetVisionProviderAsync(string? model = null, string? provider = null);
Task<IAiStructuredExtractionProvider> GetStructuredExtractionProviderAsync(string? model = null, string? provider = null);
```

Modify `api/src/Puupees.Application/Ai/Providers/DefaultAiProviderRouter.cs` to return `_mockAiProvider` for the new methods.

- [ ] **Step 8: Extend mock provider**

Modify `api/src/Puupees.Application/Ai/Providers/MockAiProvider.cs` class declaration:

```csharp
public class MockAiProvider :
    IAiChatProvider,
    IAiImageProvider,
    IAiVideoProvider,
    IAiVisionProvider,
    IAiStructuredExtractionProvider
```

Add methods:

```csharp
public Task<AiImageAnalysisDto> AnalyzeImageAsync(CreateAiImageAnalysisDto input)
{
    return Task.FromResult(new AiImageAnalysisDto
    {
        Provider = Name,
        Model = input.Model ?? "mock-vision",
        Caption = "桌面上有一台 iPhone 15 Pro、一个 Apple 充电器和一根 USB-C 数据线。",
        OcrTexts =
        {
            new AiOcrTextDto { Text = "iPhone 15 Pro", Confidence = 0.96 },
            new AiOcrTextDto { Text = "Apple", Confidence = 0.93 }
        },
        Labels =
        {
            new AiImageLabelDto { Name = "手机", Category = "电子产品", Confidence = 0.97 },
            new AiImageLabelDto { Name = "充电器", Category = "配件", Confidence = 0.88 }
        },
        Products =
        {
            new AiImageProductDto { Name = "iPhone 15 Pro", Brand = "Apple", Category = "手机", Confidence = 0.95 },
            new AiImageProductDto { Name = "USB-C 电源适配器", Brand = "Apple", Category = "配件", Confidence = 0.82 }
        },
        ProviderResults =
        {
            new AiProviderResultDto { Provider = Name, Model = "mock-vision", RequestId = $"mock_{Guid.NewGuid():N}" }
        }
    });
}

public Task<AiStructuredExtractionDto> CreateStructuredExtractionAsync(CreateAiStructuredExtractionDto input)
{
    var json = JsonDocument.Parse("""
    {
      "items": [
        {
          "name": "iPhone 15 Pro",
          "description": "通过图片识别生成的资产草稿。",
          "assetKind": "physical",
          "status": "inUse",
          "source": "camera",
          "category": "手机",
          "brand": "Apple",
          "model": "iPhone 15 Pro",
          "currency": "CNY",
          "currentValue": "6200",
          "importance": 5,
          "reminderEnabled": true,
          "tags": ["手机", "Apple"],
          "fieldConfidence": { "name": 0.96, "brand": 0.93 },
          "missingFields": ["serialNumber"]
        }
      ]
    }
    """);
    return Task.FromResult(new AiStructuredExtractionDto
    {
        Json = json,
        Confidence = 0.92,
        MissingFields = new List<string> { "serialNumber" },
        RawText = json.RootElement.GetRawText(),
        Provider = Name,
        Model = input.Model ?? "mock-extraction"
    });
}
```

- [ ] **Step 9: Add AppServices**

Create `api/src/Puupees.Application/Ai/AiVisionAppService.cs`:

```csharp
using System.Threading.Tasks;
using Microsoft.AspNetCore.Authorization;
using Puupees.Ai.Dtos;
using Puupees.Ai.Providers;
using Volo.Abp;

namespace Puupees.Ai;

[Authorize]
public class AiVisionAppService : PuupeesAppServiceBase, IAiVisionAppService
{
    private readonly IAiProviderRouter _providerRouter;

    public AiVisionAppService(IAiProviderRouter providerRouter)
    {
        _providerRouter = providerRouter;
    }

    public async Task<AiImageAnalysisDto> AnalyzeImageAsync(CreateAiImageAnalysisDto input)
    {
        if (string.IsNullOrWhiteSpace(input.Image.Url) &&
            (string.IsNullOrWhiteSpace(input.Image.Bucket) || string.IsNullOrWhiteSpace(input.Image.Key)))
        {
            throw new BusinessException("Puupees.Ai:ImageReferenceRequired");
        }
        if (input.Tasks.Count == 0)
        {
            throw new BusinessException("Puupees.Ai:ImageAnalysisTaskRequired");
        }

        var provider = await _providerRouter.GetVisionProviderAsync(input.Model, input.Provider);
        return await provider.AnalyzeImageAsync(input);
    }
}
```

Create `api/src/Puupees.Application/Ai/AiExtractionAppService.cs`:

```csharp
using System.Text.Json;
using System.Threading.Tasks;
using Microsoft.AspNetCore.Authorization;
using Puupees.Ai.Dtos;
using Puupees.Ai.Providers;
using Volo.Abp;

namespace Puupees.Ai;

[Authorize]
public class AiExtractionAppService : PuupeesAppServiceBase, IAiExtractionAppService
{
    private readonly IAiProviderRouter _providerRouter;

    public AiExtractionAppService(IAiProviderRouter providerRouter)
    {
        _providerRouter = providerRouter;
    }

    public async Task<AiStructuredExtractionDto> CreateStructuredExtractionAsync(CreateAiStructuredExtractionDto input)
    {
        if (string.IsNullOrWhiteSpace(input.SchemaName))
        {
            throw new BusinessException("Puupees.Ai:SchemaNameRequired");
        }
        if (string.IsNullOrWhiteSpace(input.JsonSchema))
        {
            throw new BusinessException("Puupees.Ai:JsonSchemaRequired");
        }

        var provider = await _providerRouter.GetStructuredExtractionProviderAsync(input.Model, input.Provider);
        var result = await provider.CreateStructuredExtractionAsync(input);
        try
        {
            _ = result.Json.RootElement.ValueKind;
        }
        catch (JsonException ex)
        {
            throw new BusinessException("Puupees.Ai:InvalidStructuredJson", innerException: ex);
        }
        return result;
    }
}
```

- [ ] **Step 10: Add controllers and route tests**

Modify `api/test/Puupees.Application.Tests/Ai/AiControllerRouteAndSecurityTests.cs`:

```csharp
[InlineData(typeof(AiVisionController), "api/ai/vision")]
[InlineData(typeof(AiExtractionController), "api/ai/extraction")]
```

Create `api/src/Puupees.HttpApi/Controllers/Ai/AiVisionController.cs`:

```csharp
using System.Threading.Tasks;
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Mvc;
using Puupees.Ai;
using Puupees.Ai.Dtos;

namespace Puupees.Controllers.Ai;

[Authorize]
[ApiController]
[Route("api/ai/vision")]
public class AiVisionController : PuupeesController
{
    private readonly IAiVisionAppService _visionAppService;

    public AiVisionController(IAiVisionAppService visionAppService)
    {
        _visionAppService = visionAppService;
    }

    [HttpPost("analysis")]
    public Task<AiImageAnalysisDto> AnalyzeImageAsync([FromBody] CreateAiImageAnalysisDto input)
    {
        return _visionAppService.AnalyzeImageAsync(input);
    }
}
```

Create `api/src/Puupees.HttpApi/Controllers/Ai/AiExtractionController.cs`:

```csharp
using System.Threading.Tasks;
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Mvc;
using Puupees.Ai;
using Puupees.Ai.Dtos;

namespace Puupees.Controllers.Ai;

[Authorize]
[ApiController]
[Route("api/ai/extraction")]
public class AiExtractionController : PuupeesController
{
    private readonly IAiExtractionAppService _extractionAppService;

    public AiExtractionController(IAiExtractionAppService extractionAppService)
    {
        _extractionAppService = extractionAppService;
    }

    [HttpPost("structured")]
    public Task<AiStructuredExtractionDto> CreateStructuredExtractionAsync(
        [FromBody] CreateAiStructuredExtractionDto input)
    {
        return _extractionAppService.CreateStructuredExtractionAsync(input);
    }
}
```

- [ ] **Step 11: Run tests and verify GREEN**

Run:

```bash
cd api
dotnet test test/Puupees.Application.Tests/Puupees.Application.Tests.csproj --filter "AiVisionAndExtractionTests|AiControllerRouteAndSecurityTests"
```

Expected: PASS.

- [ ] **Step 12: Commit**

```bash
git add api/src/Puupees.Application.Contracts/Ai api/src/Puupees.Application/Ai api/src/Puupees.HttpApi/Controllers/Ai api/test/Puupees.Application.Tests/Ai
git commit -m "feat(api): 添加通用视觉与结构化抽取接口"
```

---

### Task 4: Capability Router and Real Provider Adapters

**Files:**
- Create: `api/src/Puupees.Application/Ai/Providers/AiProviderRegistry.cs`
- Modify: `api/src/Puupees.Application/Ai/Providers/DefaultAiProviderRouter.cs`
- Create: `api/src/Puupees.Application/Ai/Providers/OpenAiCompatibleChatProvider.cs`
- Create: `api/src/Puupees.Application/Ai/Providers/TencentCloudRequestSigner.cs`
- Create: `api/src/Puupees.Application/Ai/Providers/TencentCloudVisionProvider.cs`
- Modify: `api/src/Puupees.Application/PuupeesApplicationModule.cs`
- Create: `api/test/Puupees.Application.Tests/Ai/AiProviderRouterTests.cs`
- Create: `api/test/Puupees.Application.Tests/Ai/TencentCloudProviderTests.cs`

- [ ] **Step 1: Write failing router tests**

Create `api/test/Puupees.Application.Tests/Ai/AiProviderRouterTests.cs`:

```csharp
using System.Collections.Generic;
using System.Threading.Tasks;
using Puupees.Ai;
using Puupees.Ai.Providers;
using Shouldly;
using Xunit;

namespace Puupees.Tests.Ai;

public class AiProviderRouterTests
{
    [Fact]
    public async Task Should_Use_Mock_When_No_Registry_Provider_Matches()
    {
        var mock = new MockAiProvider();
        var router = new DefaultAiProviderRouter(mock, new AiProviderRegistry());

        var provider = await router.GetVisionProviderAsync();

        provider.Name.ShouldBe("mock");
    }

    [Fact]
    public async Task Should_Route_Registered_Vision_Provider_By_Name()
    {
        var mock = new MockAiProvider();
        var registry = new AiProviderRegistry();
        registry.AddVisionProvider("tencent", mock, new[] { AiCapability.Ocr, AiCapability.ImageLabel });
        var router = new DefaultAiProviderRouter(mock, registry);

        var provider = await router.GetVisionProviderAsync(provider: "tencent");

        provider.Name.ShouldBe("mock");
    }
}
```

- [ ] **Step 2: Run tests and verify RED**

Run:

```bash
cd api
dotnet test test/Puupees.Application.Tests/Puupees.Application.Tests.csproj --filter AiProviderRouterTests
```

Expected: FAIL because `AiProviderRegistry` and router constructor overload do not exist.

- [ ] **Step 3: Implement registry**

Create `api/src/Puupees.Application/Ai/Providers/AiProviderRegistry.cs`:

```csharp
using System;
using System.Collections.Generic;
using System.Linq;

namespace Puupees.Ai.Providers;

public class AiProviderRegistry
{
    private readonly Dictionary<string, IAiVisionProvider> _visionProviders = new(StringComparer.OrdinalIgnoreCase);
    private readonly Dictionary<string, IAiStructuredExtractionProvider> _extractionProviders = new(StringComparer.OrdinalIgnoreCase);
    private readonly Dictionary<string, IAiChatProvider> _chatProviders = new(StringComparer.OrdinalIgnoreCase);
    private readonly Dictionary<string, HashSet<AiCapability>> _capabilities = new(StringComparer.OrdinalIgnoreCase);

    public void AddVisionProvider(string name, IAiVisionProvider provider, IEnumerable<AiCapability> capabilities)
    {
        _visionProviders[name] = provider;
        _capabilities[name] = capabilities.ToHashSet();
    }

    public void AddStructuredExtractionProvider(string name, IAiStructuredExtractionProvider provider, IEnumerable<AiCapability> capabilities)
    {
        _extractionProviders[name] = provider;
        _capabilities[name] = capabilities.ToHashSet();
    }

    public void AddChatProvider(string name, IAiChatProvider provider, IEnumerable<AiCapability> capabilities)
    {
        _chatProviders[name] = provider;
        _capabilities[name] = capabilities.ToHashSet();
    }

    public IAiVisionProvider? FindVisionProvider(string? name)
    {
        return name != null && _visionProviders.TryGetValue(name, out var provider) ? provider : null;
    }

    public IAiStructuredExtractionProvider? FindStructuredExtractionProvider(string? name)
    {
        return name != null && _extractionProviders.TryGetValue(name, out var provider) ? provider : null;
    }

    public IAiChatProvider? FindChatProvider(string? name)
    {
        return name != null && _chatProviders.TryGetValue(name, out var provider) ? provider : null;
    }
}
```

- [ ] **Step 4: Update router**

Modify `api/src/Puupees.Application/Ai/Providers/DefaultAiProviderRouter.cs`:

```csharp
private readonly AiProviderRegistry _registry;

public DefaultAiProviderRouter(MockAiProvider mockAiProvider)
    : this(mockAiProvider, new AiProviderRegistry())
{
}

public DefaultAiProviderRouter(MockAiProvider mockAiProvider, AiProviderRegistry registry)
{
    _mockAiProvider = mockAiProvider;
    _registry = registry;
}
```

Use registry before mock:

```csharp
public Task<IAiVisionProvider> GetVisionProviderAsync(string? model = null, string? provider = null)
{
    return Task.FromResult(_registry.FindVisionProvider(provider) ?? (IAiVisionProvider)_mockAiProvider);
}

public Task<IAiStructuredExtractionProvider> GetStructuredExtractionProviderAsync(string? model = null, string? provider = null)
{
    return Task.FromResult(_registry.FindStructuredExtractionProvider(provider) ?? (IAiStructuredExtractionProvider)_mockAiProvider);
}
```

- [ ] **Step 5: Add OpenAI-compatible provider skeleton**

Create `api/src/Puupees.Application/Ai/Providers/OpenAiCompatibleChatProvider.cs`:

```csharp
using System.Net.Http;
using System.Net.Http.Json;
using System.Text.Json;
using System.Threading.Tasks;
using Puupees.Ai.Dtos;
using Volo.Abp;

namespace Puupees.Ai.Providers;

public class OpenAiCompatibleChatProvider : IAiChatProvider, IAiStructuredExtractionProvider
{
    private readonly HttpClient _httpClient;
    public string Name => "openai-compatible";

    public OpenAiCompatibleChatProvider(HttpClient httpClient)
    {
        _httpClient = httpClient;
    }

    public async Task<AiChatCompletionDto> CreateCompletionAsync(CreateAiChatCompletionDto input)
    {
        var response = await _httpClient.PostAsJsonAsync("chat/completions", input);
        response.EnsureSuccessStatusCode();
        return await response.Content.ReadFromJsonAsync<AiChatCompletionDto>()
               ?? throw new BusinessException("Puupees.Ai:EmptyChatCompletionResponse");
    }

    public async Task<AiStructuredExtractionDto> CreateStructuredExtractionAsync(CreateAiStructuredExtractionDto input)
    {
        var chat = await CreateCompletionAsync(new CreateAiChatCompletionDto
        {
            Model = input.Model ?? "default",
            Messages =
            {
                new AiChatMessageDto
                {
                    Role = "system",
                    Content = "你只能返回合法 JSON，不要输出 Markdown。"
                },
                new AiChatMessageDto
                {
                    Role = "user",
                    Content = $"{input.Instruction}\nSchema:\n{input.JsonSchema}\nSource:\n{input.SourceText}"
                }
            }
        });
        var raw = chat.Choices.Count > 0 ? chat.Choices[0].Message.Content : "{}";
        return new AiStructuredExtractionDto
        {
            Json = JsonDocument.Parse(raw),
            Confidence = 1,
            RawText = raw,
            Provider = Name,
            Model = chat.Model,
            Usage = chat.Usage
        };
    }
}
```

- [ ] **Step 6: Add Tencent signer and provider skeleton**

Create `api/src/Puupees.Application/Ai/Providers/TencentCloudRequestSigner.cs`:

```csharp
using System;
using System.Security.Cryptography;
using System.Text;

namespace Puupees.Ai.Providers;

public static class TencentCloudRequestSigner
{
    public static string Sha256Hex(string payload)
    {
        var hash = SHA256.HashData(Encoding.UTF8.GetBytes(payload));
        return Convert.ToHexString(hash).ToLowerInvariant();
    }
}
```

Create `api/src/Puupees.Application/Ai/Providers/TencentCloudVisionProvider.cs`:

```csharp
using System.Threading.Tasks;
using Puupees.Ai.Dtos;

namespace Puupees.Ai.Providers;

public class TencentCloudVisionProvider : IAiVisionProvider
{
    public string Name => "tencent-cloud";

    public Task<AiImageAnalysisDto> AnalyzeImageAsync(CreateAiImageAnalysisDto input)
    {
        return Task.FromResult(new AiImageAnalysisDto
        {
            Provider = Name,
            Model = input.Model ?? "tencent-cloud-vision",
            Caption = "腾讯云视觉识别结果摘要。",
            ProviderResults =
            {
                new AiProviderResultDto { Provider = Name, Model = "tencent-cloud-vision" }
            }
        });
    }
}
```

- [ ] **Step 7: Add Tencent provider unit test**

Create `api/test/Puupees.Application.Tests/Ai/TencentCloudProviderTests.cs`:

```csharp
using Puupees.Ai.Providers;
using Shouldly;
using Xunit;

namespace Puupees.Tests.Ai;

public class TencentCloudProviderTests
{
    [Fact]
    public void Should_Create_Sha256_Hex_For_Request_Payload()
    {
        TencentCloudRequestSigner.Sha256Hex("{}")
            .ShouldBe("44136fa355b3678a1146ad16f7e8649e94fb4fc21fe77e8310c060f61caaff8a");
    }
}
```

- [ ] **Step 8: Register providers**

Modify `api/src/Puupees.Application/PuupeesApplicationModule.cs`:

```csharp
context.Services.AddSingleton<Puupees.Ai.Providers.AiProviderRegistry>();
context.Services.AddHttpClient<Puupees.Ai.Providers.OpenAiCompatibleChatProvider>();
context.Services.AddSingleton<Puupees.Ai.Providers.TencentCloudVisionProvider>();
```

Keep existing mock registration.

- [ ] **Step 9: Run tests and verify GREEN**

Run:

```bash
cd api
dotnet test test/Puupees.Application.Tests/Puupees.Application.Tests.csproj --filter "AiProviderRouterTests|TencentCloudProviderTests"
```

Expected: PASS.

- [ ] **Step 10: Commit**

```bash
git add api/src/Puupees.Application/Ai/Providers api/src/Puupees.Application/PuupeesApplicationModule.cs api/test/Puupees.Application.Tests/Ai
git commit -m "feat(api): 添加 AI 能力路由和服务商适配器"
```

---

### Task 5: API Verification and OpenAPI Surface

**Files:**
- Modify: `api/test/Puupees.Application.Tests/Ai/AiGatewayAppServiceTests.cs`
- Modify: `api/test/Puupees.Application.Tests/Ai/AiControllerRouteAndSecurityTests.cs`
- Modify: `api/src/Puupees.HttpApi/Controllers/Ai/*.cs`

- [ ] **Step 1: Run full API AI tests**

Run:

```bash
cd api
dotnet test test/Puupees.Application.Tests/Puupees.Application.Tests.csproj --filter "FullyQualifiedName~Ai"
```

Expected: PASS.

- [ ] **Step 2: Run API build**

Run:

```bash
cd api
dotnet build Puupees.sln
```

Expected: build succeeds with no compile errors.

- [ ] **Step 3: Commit fixes if needed**

If Step 1 or Step 2 required compile fixes, stage only touched API files:

```bash
git add api/src api/test
git commit -m "fix(api): 完善 AI 通用接口编译与测试"
```

Expected: commit only when there are actual fixes.

---

### Task 6: Regenerate Dart API Client

**Files:**
- Generated/Modify: `packages/api/felorx_api_client/lib/**`
- Generated/Modify: `packages/api/felorx_api_client/doc/**`
- Generated/Modify: `packages/api/felorx_api_client/test/**`
- Modify: `packages/api/felorx_api_client/pubspec.yaml` only if generator updates it.

- [ ] **Step 1: Start API host for swagger**

Run:

```bash
cd api
dotnet run --project src/Puupees.HttpApi.Host
```

Expected: host starts and exposes Swagger JSON. Keep the terminal running for the next step.

- [ ] **Step 2: Regenerate Dart client**

In a second terminal:

```bash
cd packages/api/puupee_sdk_generator
dart run bin/puupee_sdk_generator.dart dart --swagger-url http://localhost:5000/swagger/v1/swagger.json
```

Expected: `packages/api/felorx_api_client` gains APIs/models for `AiProvidersApi`, `AiVisionApi`, and `AiExtractionApi`.

- [ ] **Step 3: Run client code generation**

Run:

```bash
cd packages/api/felorx_api_client
dart run build_runner build --delete-conflicting-outputs
```

Expected: generated `*.g.dart` files are updated successfully.

- [ ] **Step 4: Verify generated client analyze**

Run:

```bash
cd packages/api/felorx_api_client
dart analyze
```

Expected: analysis succeeds.

- [ ] **Step 5: Commit**

```bash
git add packages/api/felorx_api_client
git commit -m "chore(api-client): 生成 AI 通用能力客户端"
```

---

### Task 7: Developer AI Provider Management UI

**Files:**
- Modify: `apps/developer/lib/development/home_page.dart`
- Modify: `apps/developer/lib/router.dart`
- Create: `apps/developer/lib/ai-providers/home_page.dart`
- Create: `apps/developer/lib/ai-providers/list_view.dart`
- Create: `apps/developer/lib/ai-providers/list_view_item.dart`
- Create: `apps/developer/lib/ai-providers/edit_page.dart`
- Create: `apps/developer/lib/ai-providers/provider.dart`

- [ ] **Step 1: Write failing provider unit smoke test**

Create `apps/developer/test/ai_provider_provider_test.dart`:

```dart
import 'package:flutter_test/flutter_test.dart';
import 'package:puupee_developer/ai-providers/provider.dart';

void main() {
  test('AI provider page size stays aligned with existing list pages', () {
    expect(aiProviderPageSize, 10);
  });
}
```

Run:

```bash
cd apps/developer
flutter test test/ai_provider_provider_test.dart
```

Expected: FAIL because `ai-providers/provider.dart` does not exist.

- [ ] **Step 2: Add provider file**

Create `apps/developer/lib/ai-providers/provider.dart`:

```dart
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:felorx_api_client/felorx_api_client.dart';
import 'package:puupee_shared/providers/base/api_client.dart';

const aiProviderPageSize = 10;

final paginatedAiProvidersProvider =
    FutureProvider.family<AiProviderDtoPagedResultDto, int>((ref, pageIndex) async {
  final apiClient = ref.read(apiClientProvider);
  final result = await apiClient.getAiProvidersApi().getList(
        skipCount: pageIndex * aiProviderPageSize,
        maxResultCount: aiProviderPageSize,
      );
  return result.data ?? AiProviderDtoPagedResultDto();
});

final aiProvidersCountProvider = Provider<AsyncValue<int>>((ref) {
  return ref.watch(paginatedAiProvidersProvider(0)).whenData(
        (pageData) => pageData.totalCount ?? 0,
      );
});
```

- [ ] **Step 3: Add list item widget**

Create `apps/developer/lib/ai-providers/list_view_item.dart`:

```dart
import 'package:felorx_api_client/felorx_api_client.dart';
import 'package:shadcn_flutter/shadcn_flutter.dart';

class AiProviderListViewItem extends StatelessWidget {
  const AiProviderListViewItem({
    super.key,
    required this.provider,
    required this.onEdit,
  });

  final AiProviderDto provider;
  final VoidCallback onEdit;

  @override
  Widget build(BuildContext context) {
    final theme = Theme.of(context);
    final capabilities = provider.capabilities ?? const <AiCapability>[];
    return SurfaceCard(
      child: ListTile(
        title: Text(provider.displayName ?? provider.name ?? ''),
        subtitle: Text(provider.providerType?.name ?? ''),
        trailing: Row(
          mainAxisSize: MainAxisSize.min,
          children: [
            for (final capability in capabilities.take(3))
              Padding(
                padding: const EdgeInsets.only(left: 4),
                child: Badge(child: Text(capability.name)),
              ),
            const SizedBox(width: 8),
            Icon(
              provider.enabled == true ? Icons.check_circle : Icons.pause_circle,
              color: provider.enabled == true
                  ? theme.colorScheme.primary
                  : theme.colorScheme.mutedForeground,
            ),
            IconButton.ghost(icon: const Icon(Icons.edit), onPressed: onEdit),
          ],
        ),
      ),
    );
  }
}
```

- [ ] **Step 4: Add list page**

Create `apps/developer/lib/ai-providers/list_view.dart`:

```dart
import 'package:flutter/cupertino.dart' show CupertinoPageRoute;
import 'package:hooks_riverpod/hooks_riverpod.dart';
import 'package:puupee_developer/ai-providers/edit_page.dart';
import 'package:puupee_developer/ai-providers/list_view_item.dart';
import 'package:puupee_developer/ai-providers/provider.dart';
import 'package:shadcn_flutter/shadcn_flutter.dart';

class AiProviderListView extends ConsumerWidget {
  const AiProviderListView({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final page = ref.watch(paginatedAiProvidersProvider(0));
    return page.when(
      loading: () => const Center(child: CircularProgressIndicator()),
      error: (error, stackTrace) => Center(child: Text('加载失败: $error')),
      data: (data) {
        final items = data.items ?? const [];
        if (items.isEmpty) {
          return const Center(child: Text('暂无 AI 服务商'));
        }
        return ListView.separated(
          padding: const EdgeInsets.all(16),
          itemCount: items.length,
          separatorBuilder: (_, __) => const SizedBox(height: 8),
          itemBuilder: (context, index) {
            final item = items[index];
            return AiProviderListViewItem(
              provider: item,
              onEdit: () {
                Navigator.of(context).push(
                  CupertinoPageRoute<void>(
                    builder: (_) => AiProviderEditPage(provider: item),
                  ),
                );
              },
            );
          },
        );
      },
    );
  }
}
```

- [ ] **Step 5: Add edit page**

Create `apps/developer/lib/ai-providers/edit_page.dart`:

```dart
import 'package:flutter/material.dart' as material;
import 'package:go_router/go_router.dart';
import 'package:hooks_riverpod/hooks_riverpod.dart';
import 'package:felorx_api_client/felorx_api_client.dart';
import 'package:puupee_developer/ai-providers/provider.dart';
import 'package:puupee_shared/providers/base/api_client.dart';
import 'package:puupee_shared/utils/toast.dart';
import 'package:puupee_ui/components/adaptive_scaffold.dart';
import 'package:reactive_forms/reactive_forms.dart';
import 'package:shadcn_flutter/shadcn_flutter.dart';

class AiProviderEditPage extends ConsumerStatefulWidget {
  const AiProviderEditPage({super.key, this.provider});

  final AiProviderDto? provider;

  @override
  ConsumerState<AiProviderEditPage> createState() => _AiProviderEditPageState();
}

class _AiProviderEditPageState extends ConsumerState<AiProviderEditPage> {
  late final FormGroup form;

  @override
  void initState() {
    super.initState();
    final provider = widget.provider;
    form = FormGroup({
      'name': FormControl<String>(value: provider?.name, validators: [Validators.required]),
      'displayName': FormControl<String>(value: provider?.displayName, validators: [Validators.required]),
      'providerType': FormControl<String>(value: provider?.providerType?.name ?? AiProviderType.openAiCompatible.name),
      'baseUrl': FormControl<String>(value: provider?.baseUrl),
      'region': FormControl<String>(value: provider?.region),
      'enabled': FormControl<bool>(value: provider?.enabled ?? true),
      'secret': FormControl<String>(),
    });
  }

  Future<void> _save() async {
    final api = ref.read(apiClientProvider).getAiProvidersApi();
    final value = form.value;
    final dto = CreateOrUpdateAiProviderDto(
      name: value['name'] as String?,
      displayName: value['displayName'] as String?,
      providerType: AiProviderType.values.firstWhere(
        (type) => type.name == value['providerType'],
      ),
      baseUrl: value['baseUrl'] as String?,
      region: value['region'] as String?,
      enabled: value['enabled'] as bool?,
      secret: value['secret'] as String?,
      capabilities: const [
        AiCapability.chat,
        AiCapability.structuredExtraction,
      ],
      models: const [],
    );

    if (widget.provider?.id == null) {
      await api.create(createOrUpdateAiProviderDto: dto);
    } else {
      await api.update(
        id: widget.provider!.id!,
        createOrUpdateAiProviderDto: dto,
      );
    }

    ref.invalidate(paginatedAiProvidersProvider);
    if (mounted) {
      await showMessage('AI 服务商已保存');
      context.pop();
    }
  }

  @override
  Widget build(BuildContext context) {
    return AdaptiveScaffold(
      showLeading: true,
      title: widget.provider == null ? '新增 AI 服务商' : '编辑 AI 服务商',
      body: ReactiveForm(
        formGroup: form,
        child: ListView(
          padding: const EdgeInsets.all(24),
          children: [
            ReactiveTextField(formControlName: 'name', decoration: const material.InputDecoration(labelText: '名称')),
            ReactiveTextField(formControlName: 'displayName', decoration: const material.InputDecoration(labelText: '显示名称')),
            ReactiveDropdownField<String>(
              formControlName: 'providerType',
              decoration: const material.InputDecoration(labelText: '类型'),
              items: AiProviderType.values
                  .map((type) => material.DropdownMenuItem(value: type.name, child: Text(type.name)))
                  .toList(),
            ),
            ReactiveTextField(formControlName: 'baseUrl', decoration: const material.InputDecoration(labelText: 'Base URL')),
            ReactiveTextField(formControlName: 'region', decoration: const material.InputDecoration(labelText: 'Region')),
            ReactiveTextField(formControlName: 'secret', decoration: const material.InputDecoration(labelText: '密钥')),
            const SizedBox(height: 24),
            ReactiveFormConsumer(
              builder: (context, form, child) => PrimaryButton(
                onPressed: form.valid ? _save : null,
                child: const Text('保存'),
              ),
            ),
          ],
        ),
      ),
    );
  }
}
```

- [ ] **Step 6: Add home page**

Create `apps/developer/lib/ai-providers/home_page.dart`:

```dart
import 'package:flutter/cupertino.dart' show CupertinoPageRoute;
import 'package:hooks_riverpod/hooks_riverpod.dart';
import 'package:puupee_developer/ai-providers/edit_page.dart';
import 'package:puupee_developer/ai-providers/list_view.dart';
import 'package:puupee_ui/components/adaptive_scaffold.dart';
import 'package:puupee_ui/components/responsive_builder.dart';
import 'package:shadcn_flutter/shadcn_flutter.dart';

class AiProvidersPage extends ConsumerWidget {
  const AiProvidersPage({super.key});

  void _handleAdd(BuildContext context) {
    Navigator.of(context).push(
      CupertinoPageRoute<void>(builder: (_) => const AiProviderEditPage()),
    );
  }

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    return ResponsiveBuilder(
      builder: (sizingInfo) {
        return AdaptiveScaffold(
          showLeading: true,
          title: 'AI 服务商',
          body: const AiProviderListView(),
          actions: sizingInfo.isTabletOrDesktop
              ? [
                  Tooltip(
                    tooltip: (_) => const Text('添加 AI 服务商'),
                    child: IconButton.ghost(
                      icon: const Icon(Icons.add),
                      onPressed: () => _handleAdd(context),
                    ),
                  ),
                ]
              : null,
          floatingActionButton: sizingInfo.isMobile
              ? PrimaryButton(
                  onPressed: () => _handleAdd(context),
                  child: const Icon(Icons.add),
                )
              : null,
        );
      },
    );
  }
}
```

- [ ] **Step 7: Add development entry and route**

Modify `apps/developer/lib/development/home_page.dart`:

```dart
import '../ai-providers/home_page.dart';
```

Add a third navigation card after API Key:

```dart
_buildNavigationCard(
  context: context,
  title: 'AI 服务商',
  subtitle: '管理 AI 服务提供商和模型',
  icon: Icons.auto_awesome,
  color: Colors.purple,
  onTap: () {
    Navigator.of(context).push(
      CupertinoPageRoute<void>(
        builder: (context) => const AiProvidersPage(),
      ),
    );
  },
),
```

Modify `apps/developer/lib/router.dart`:

```dart
import 'ai-providers/home_page.dart';
```

Add route:

```dart
TypedGoRoute<DeveloperAiProvidersRoute>(
  path: '/developer/development/ai-providers',
),
```

Add route data:

```dart
class DeveloperAiProvidersRoute extends GoRouteData {
  const DeveloperAiProvidersRoute();

  @override
  Widget build(BuildContext context, GoRouterState state) {
    return const AuthGuard(showLoginDialog: true, child: AiProvidersPage());
  }
}
```

- [ ] **Step 8: Generate router code**

Run:

```bash
cd apps/developer
dart run build_runner build --delete-conflicting-outputs
```

Expected: `router.g.dart` updates.

- [ ] **Step 9: Run developer tests/analyze**

Run:

```bash
cd apps/developer
flutter test test/ai_provider_provider_test.dart
dart analyze .
```

Expected: test and analysis pass.

- [ ] **Step 10: Commit**

```bash
git add apps/developer
git commit -m "feat(developer): 添加 AI 服务商管理入口"
```

---

### Task 8: Inventory API Recognition Service Mapping

**Files:**
- Create: `apps/inventory/lib/services/api_inventory_recognition_service.dart`
- Create: `apps/inventory/test/api_inventory_recognition_service_test.dart`
- Modify: `apps/inventory/lib/services/inventory_recognition_service.dart`

- [ ] **Step 1: Write failing mapping test**

Create `apps/inventory/test/api_inventory_recognition_service_test.dart`:

```dart
import 'package:flutter_test/flutter_test.dart';
import 'package:puupee/puupee.dart';
import 'package:puupee_inventory/services/api_inventory_recognition_service.dart';

void main() {
  test('maps structured extraction item to inventory draft', () {
    final draft = inventoryDraftFromStructuredJson({
      'name': 'iPhone 15 Pro',
      'description': '识别自桌面照片',
      'assetKind': 'physical',
      'status': 'inUse',
      'source': 'camera',
      'category': '手机',
      'brand': 'Apple',
      'model': 'iPhone 15 Pro',
      'currency': 'CNY',
      'currentValue': '6200',
      'importance': 5,
      'reminderEnabled': true,
      'tags': ['手机', 'Apple'],
      'fieldConfidence': {'name': 0.96},
      'missingFields': ['serialNumber'],
    });

    expect(draft.name, 'iPhone 15 Pro');
    expect(draft.assetKind, InventoryAssetKind.physical);
    expect(draft.status, InventoryAssetStatus.inUse);
    expect(draft.source, InventoryAssetSource.camera);
    expect(draft.brand, 'Apple');
    expect(draft.currentValue.toString(), '6200');
    expect(draft.fieldConfidence['name'], 0.96);
    expect(draft.missingFields, contains('serialNumber'));
  });
}
```

- [ ] **Step 2: Run test and verify RED**

Run:

```bash
cd apps/inventory
flutter test test/api_inventory_recognition_service_test.dart
```

Expected: FAIL because `api_inventory_recognition_service.dart` does not exist.

- [ ] **Step 3: Add mapping helper and service skeleton**

Create `apps/inventory/lib/services/api_inventory_recognition_service.dart`:

```dart
import 'dart:io';

import 'package:decimal/decimal.dart';
import 'package:puupee/puupee.dart';
import 'package:felorx_api_client/felorx_api_client.dart';

import '../models/inventory_asset_draft.dart';
import 'inventory_recognition_service.dart';

class ApiInventoryRecognitionService implements InventoryRecognitionService {
  const ApiInventoryRecognitionService({
    required this.apiClient,
    required this.uploadImage,
  });

  final FelorxApiClient apiClient;
  final Future<AiMediaReferenceDto> Function(File file) uploadImage;

  @override
  Future<List<InventoryAssetDraft>> recognizePhysicalPhoto({String? imagePath}) {
    return _recognize(imagePath: imagePath, source: InventoryAssetSource.camera);
  }

  @override
  Future<List<InventoryAssetDraft>> recognizeVirtualScreenshot({String? imagePath}) {
    return _recognize(imagePath: imagePath, source: InventoryAssetSource.screenshot);
  }

  Future<List<InventoryAssetDraft>> _recognize({
    required String? imagePath,
    required InventoryAssetSource source,
  }) async {
    if (imagePath == null || imagePath.trim().isEmpty) {
      return const [];
    }
    final image = await uploadImage(File(imagePath));
    final vision = await apiClient.getAiVisionApi().analyzeImage(
          createAiImageAnalysisDto: CreateAiImageAnalysisDto(
            image: image,
            tasks: source == InventoryAssetSource.camera
                ? const [
                    AiImageAnalysisTask.products,
                    AiImageAnalysisTask.labels,
                    AiImageAnalysisTask.ocr,
                    AiImageAnalysisTask.caption,
                  ]
                : const [
                    AiImageAnalysisTask.ocr,
                    AiImageAnalysisTask.labels,
                    AiImageAnalysisTask.caption,
                  ],
            locale: 'zh-CN',
          ),
        );
    final extraction = await apiClient.getAiExtractionApi().createStructuredExtraction(
          createAiStructuredExtractionDto: CreateAiStructuredExtractionDto(
            schemaName: 'inventory_asset_draft',
            jsonSchema: inventoryAssetDraftJsonSchema,
            sourceVisionResult: vision.data,
            instruction: source == InventoryAssetSource.camera
                ? '从图片分析结果中提取实物资产草稿。'
                : '从截图 OCR 和图片分析结果中提取虚拟资产或订阅草稿。',
          ),
        );
    final root = extraction.data?.json;
    final items = root?['items'];
    if (items is! List) {
      return const [];
    }
    return items
        .whereType<Map<String, dynamic>>()
        .map(inventoryDraftFromStructuredJson)
        .toList();
  }
}

const inventoryAssetDraftJsonSchema = '''
{"type":"object","properties":{"items":{"type":"array","items":{"type":"object"}}},"required":["items"]}
''';

InventoryAssetDraft inventoryDraftFromStructuredJson(Map<String, dynamic> json) {
  return InventoryAssetDraft(
    name: json['name'] as String?,
    description: json['description'] as String?,
    assetKind: _enumByName(InventoryAssetKind.values, json['assetKind']) ?? InventoryAssetKind.physical,
    status: _enumByName(InventoryAssetStatus.values, json['status']) ?? InventoryAssetStatus.inUse,
    source: _enumByName(InventoryAssetSource.values, json['source']),
    category: json['category'] as String?,
    brand: json['brand'] as String?,
    model: json['model'] as String?,
    serialNumber: json['serialNumber'] as String?,
    location: json['location'] as String?,
    accountHint: json['accountHint'] as String?,
    planName: json['planName'] as String?,
    currency: json['currency'] as String?,
    purchasePrice: _decimal(json['purchasePrice']),
    currentValue: _decimal(json['currentValue']),
    purchaseDate: _date(json['purchaseDate']),
    warrantyEndDate: _date(json['warrantyEndDate']),
    renewalDate: _date(json['renewalDate']),
    billingCycle: _enumByName(InventoryBillingCycle.values, json['billingCycle']),
    importance: json['importance'] as int?,
    reminderEnabled: json['reminderEnabled'] as bool?,
    tags: (json['tags'] as List?)?.whereType<String>().toList() ?? const [],
    fieldConfidence: (json['fieldConfidence'] as Map?)?.map(
          (key, value) => MapEntry(key.toString(), (value as num).toDouble()),
        ) ??
        const {},
    missingFields: (json['missingFields'] as List?)?.map((value) => value.toString()).toSet() ?? const {},
  );
}

T? _enumByName<T extends Enum>(List<T> values, Object? value) {
  final name = value?.toString();
  if (name == null || name.isEmpty) return null;
  for (final item in values) {
    if (item.name == name) return item;
  }
  return null;
}

Decimal? _decimal(Object? value) {
  if (value == null) return null;
  return Decimal.tryParse(value.toString());
}

DateTime? _date(Object? value) {
  if (value == null || value.toString().isEmpty) return null;
  return DateTime.tryParse(value.toString());
}
```

- [ ] **Step 4: Run mapping test and fix generated model access**

Run:

```bash
cd apps/inventory
flutter test test/api_inventory_recognition_service_test.dart
```

Expected: PASS after adapting `root?['items']` to the actual generated `AiStructuredExtractionDto` JSON field shape if the generator uses `Map<String, dynamic>` instead of `JsonDocument`.

- [ ] **Step 5: Commit**

```bash
git add apps/inventory/lib/services/api_inventory_recognition_service.dart apps/inventory/test/api_inventory_recognition_service_test.dart
git commit -m "feat(inventory): 添加通用 AI 识别结果映射"
```

---

### Task 9: Inventory Upload and Provider Wiring

**Files:**
- Create: `apps/inventory/lib/services/inventory_image_upload_service.dart`
- Modify: `apps/inventory/lib/providers/inventory_providers.dart`
- Modify: `apps/inventory/lib/providers/inventory_providers.g.dart`
- Modify: `apps/inventory/test/inventory_recognition_service_test.dart`
- Modify: `apps/inventory/test/recognition_confirm_page_test.dart`

- [ ] **Step 1: Add image upload service**

Create `apps/inventory/lib/services/inventory_image_upload_service.dart`:

```dart
import 'dart:io';

import 'package:mime/mime.dart';
import 'package:path/path.dart' as path;
import 'package:felorx_api_client/felorx_api_client.dart';
import 'package:puupee_shared/components/file_upload/upload_provider.dart';
import 'package:riverpod_annotation/riverpod_annotation.dart';

part 'inventory_image_upload_service.g.dart';

class InventoryImageUploadService {
  const InventoryImageUploadService(this.uploadFile);

  final Future<String> Function(File file, FileUploadConfig config) uploadFile;

  Future<AiMediaReferenceDto> uploadRecognitionImage(File file) async {
    final key = await uploadFile(
      file,
      const FileUploadConfig(
        usage: 'Files',
        keyPrefix: 'inventory/recognition/',
        addToPuupee: true,
      ),
    );
    return AiMediaReferenceDto(
      key: key,
      mimeType: lookupMimeType(path.basename(file.path)) ?? 'image/jpeg',
    );
  }
}

@riverpod
InventoryImageUploadService inventoryImageUploadService(Ref ref) {
  return InventoryImageUploadService(
    (file, config) => ref.read(fileUploadProvider.notifier).uploadFile(file, config),
  );
}
```

- [ ] **Step 2: Wire real recognition service**

Modify `apps/inventory/lib/providers/inventory_providers.dart`:

```dart
import 'package:puupee_inventory/services/api_inventory_recognition_service.dart';
import 'package:puupee_inventory/services/inventory_image_upload_service.dart';
import 'package:puupee_shared/providers/base/api_client.dart';
```

Replace provider body:

```dart
@riverpod
InventoryRecognitionService inventoryRecognitionService(Ref ref) {
  final apiClient = ref.watch(apiClientProvider);
  final uploadService = ref.watch(inventoryImageUploadServiceProvider);
  return ApiInventoryRecognitionService(
    apiClient: apiClient,
    uploadImage: uploadService.uploadRecognitionImage,
  );
}
```

- [ ] **Step 3: Regenerate inventory providers**

Run:

```bash
cd apps/inventory
dart run build_runner build --delete-conflicting-outputs
```

Expected: `inventory_providers.g.dart` and `inventory_image_upload_service.g.dart` are generated.

- [ ] **Step 4: Keep mock tests explicit**

Modify `apps/inventory/test/inventory_recognition_service_test.dart` to test `MockInventoryRecognitionService` directly, not the Riverpod provider:

```dart
const service = MockInventoryRecognitionService();
```

Existing expectations remain.

- [ ] **Step 5: Run inventory tests**

Run:

```bash
cd apps/inventory
flutter test test/inventory_recognition_service_test.dart test/api_inventory_recognition_service_test.dart test/recognition_confirm_page_test.dart
dart analyze .
```

Expected: tests and analysis pass.

- [ ] **Step 6: Commit**

```bash
git add apps/inventory
git commit -m "feat(inventory): 使用通用 AI 接口识别图片"
```

---

### Task 10: Final Verification

**Files:**
- No new files unless verification reveals fixes.

- [ ] **Step 1: Check git status**

Run:

```bash
git status --short
```

Expected: only unrelated pre-existing `vela` modification remains, or the tree is clean if the user handled it separately.

- [ ] **Step 2: Run focused API tests**

Run:

```bash
cd api
dotnet test test/Puupees.Application.Tests/Puupees.Application.Tests.csproj --filter "FullyQualifiedName~Ai"
```

Expected: PASS.

- [ ] **Step 3: Run Flutter focused tests**

Run:

```bash
cd apps/developer
flutter test test/ai_provider_provider_test.dart
dart analyze .
cd ../inventory
flutter test test/inventory_recognition_service_test.dart test/api_inventory_recognition_service_test.dart test/recognition_confirm_page_test.dart
dart analyze .
```

Expected: PASS.

- [ ] **Step 4: Run repository-wide analysis after focused checks pass**

Run when time allows:

```bash
cd /Users/j/repos/puupees/puupee-apps
melos run analyze
```

Expected: no new analysis errors caused by this feature.

- [ ] **Step 5: Commit verification fixes**

If verification required small fixes:

```bash
git add api apps/developer apps/inventory packages/api/felorx_api_client
git commit -m "fix(ai): 完成通用视觉能力验证"
```

Expected: commit only when there are actual fixes.
