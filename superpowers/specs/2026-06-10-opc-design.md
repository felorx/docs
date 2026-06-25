# opc - One Person Company 智能体工作台设计文档

**创建日期：** 2026-06-10
**版本：** 1.0
**状态：** 已批准

## 1. 概述

opc（One Person Company）是一个应用版的 OpenClaw，本质上是一个智能体工作台。主界面是一个公司的平面布局视图，有不同工位，工位上可以安排对应的员工（智能体/skill）。

### 1.1 核心目标

- **智能体工作台** — 主要用来管理和调度 AI 智能体，工位代表不同的工作角色
- **预设角色** — 提供固定的智能体角色（如"程序员"、"设计师"、"产品经理"），每个角色有预设的技能和行为
- **动态布局** — 根据智能体数量和类型自动生成最优布局，用户无需手动安排
- **统一任务面板** — 用户创建任务后系统自动分配给合适的智能体
- **完全协作** — 智能体可以自由通信，共同处理复杂任务

### 1.2 技术选型

| 类别 | 选择 | 说明 |
|------|------|------|
| 状态管理 | Riverpod | 与现有项目一致，响应式特性适合智能体状态更新 |
| 路由 | Go Router | 类型化路由，支持 Shell 路由 |
| UI 组件 | shadcn_flutter | 扁平现代设计风格 |
| 数据同步 | PuupeeSync | 复用现有同步架构，支持多设备同步 |
| AI 服务 | 云端 API | 调用 OpenAI、Claude 等云端 AI 服务 |
| 目标平台 | 桌面端优先 | macOS、Windows、Linux 为主，移动端为辅 |

## 2. 应用架构

### 2.1 整体架构

```
┌─────────────────────────────────────┐
│           UI 层 (shadcn_flutter)    │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐│
│  │ 主界面  │ │ 工位视图│ │任务面板 ││
│  └─────────┘ └─────────┘ └─────────┘│
├─────────────────────────────────────┤
│           状态层 (Riverpod)         │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐│
│  │智能体状态│ │布局状态 │ │任务状态 ││
│  └─────────┘ └─────────┘ └─────────┘│
├─────────────────────────────────────┤
│           业务层                     │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐│
│  │智能体管理│ │布局引擎 │ │任务分配 ││
│  └─────────┘ └─────────┘ └─────────┘│
├─────────────────────────────────────┤
│           数据层                     │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐│
│  │PuupeeSync│ │本地数据库│ │云端 API ││
│  └─────────┘ └─────────┘ └─────────┘│
└─────────────────────────────────────┘
```

### 2.2 目录结构

```
apps/opc/
├── lib/
│   ├── main.dart              # 应用入口
│   ├── env.dart               # 环境配置
│   ├── router.dart            # 路由定义
│   ├── provider.dart          # 全局 provider
│   ├── pages/
│   │   ├── home/              # 主页面
│   │   ├── workspace/         # 工作区页面
│   │   ├── agents/            # 智能体管理
│   │   └── settings/          # 设置页面
│   ├── components/
│   │   ├── workstation/       # 工位组件
│   │   ├── agent_card/        # 智能体卡片
│   │   ├── task_panel/        # 任务面板
│   │   └── layout/            # 布局组件
│   ├── providers/
│   │   ├── agents/            # 智能体状态
│   │   ├── layout/            # 布局状态
│   │   ├── tasks/             # 任务状态
│   │   └── workspace/         # 工作区状态
│   ├── models/
│   │   ├── agent.dart         # 智能体模型
│   │   ├── workstation.dart   # 工位模型
│   │   ├── task.dart          # 任务模型
│   │   └── layout.dart        # 布局模型
│   ├── repositories/
│   │   ├── agent_repo.dart    # 智能体仓库
│   │   ├── task_repo.dart     # 任务仓库
│   │   └── layout_repo.dart   # 布局仓库
│   └── services/
│       ├── ai_service.dart    # AI 服务
│       ├── layout_engine.dart # 布局引擎
│       └── task_allocator.dart # 任务分配器
├── assets/
│   ├── icons/                 # 智能体图标
│   └── layouts/               # 预设布局
└── test/
    ├── providers/             # provider 测试
    ├── repositories/          # 仓库测试
    └── services/              # 服务测试
```

### 2.3 核心依赖

**共享包：**
- `puupee_shared` — 共享组件和工具
- `puupee_ui` — UI 组件库
- `puupee_sync` — 数据同步
- `felorx_api_client` — API 客户端

**状态管理：**
- `flutter_riverpod` — 响应式状态管理
- `riverpod_annotation` — 代码生成注解
- `riverpod_generator` — 代码生成器

**UI 组件：**
- `shadcn_flutter` — 扁平现代 UI 组件
- `go_router` — 类型化路由
- `flutter_svg` — SVG 支持

**数据同步：**
- `drift` — 本地数据库
- `dio` — HTTP 客户端
- `freezed` — 不可变数据类

## 3. 智能体系统

### 3.1 预设智能体角色

| 角色 | 图标 | 技能 | 职责 |
|------|------|------|------|
| 程序员 | 👨‍💻 | Flutter, Dart, 代码编写 | 代码编写、调试、重构 |
| 设计师 | 🎨 | Figma, UI/UX, 设计 | UI/UX 设计、原型制作 |
| 产品经理 | 📋 | 需求分析, 功能规划 | 需求分析、功能规划 |
| 数据分析师 | 📊 | 数据处理, 报告生成 | 数据处理、报告生成 |
| 内容创作者 | ✍️ | 文案撰写, 内容策划 | 文案撰写、内容策划 |
| 市场专员 | 📈 | 市场调研, 营销策略 | 市场调研、营销策略 |

### 3.2 智能体状态管理

**状态流转：**
```
空闲 → 工作中 → 协作中 → 完成
```

**数据模型：**
```dart
@freezed
class Agent with _$Agent {
  const factory Agent({
    required String id,
    required String name,           // 智能体名称
    required AgentRole role,        // 角色类型
    required AgentStatus status,    // 当前状态
    required List<String> skills,   // 技能列表
    String? currentTaskId,          // 当前任务ID
    Map<String, dynamic>? config,   // 配置信息
    DateTime? lastActiveAt,         // 最后活跃时间
  }) = _Agent;
}

enum AgentRole {
  programmer,    // 程序员
  designer,      // 设计师
  productManager,// 产品经理
  dataAnalyst,   // 数据分析师
  contentCreator,// 内容创作者
  marketer,      // 市场专员
}

enum AgentStatus {
  idle,          // 空闲
  working,       // 工作中
  collaborating, // 协作中
  completed,     // 完成
  error,         // 错误
}
```

### 3.3 智能体通信机制

**完全协作模式：**
- **消息总线** — 智能体通过消息总线进行异步通信，支持广播和点对点消息
- **共享工作区** — 智能体可以访问共享的工作区，交换文件和数据
- **任务依赖** — 支持任务依赖关系，一个智能体的输出可以作为另一个的输入

## 4. 布局系统

### 4.1 动态布局生成

**生成算法：**
1. **分析智能体集合** — 根据智能体的角色、技能和数量，确定布局需求
2. **计算最优布局** — 使用算法计算工位位置，确保相关角色靠近，减少协作距离
3. **生成工位** — 根据计算结果生成工位，分配智能体到对应位置
4. **保存布局** — 将布局信息保存到本地，支持下次启动时恢复

### 4.2 布局数据模型

```dart
@freezed
class WorkspaceLayout with _$WorkspaceLayout {
  const factory WorkspaceLayout({
    required String id,
    required String name,           // 布局名称
    required List<Workstation> stations, // 工位列表
    required LayoutType type,       // 布局类型
    DateTime? lastUsedAt,           // 最后使用时间
  }) = _WorkspaceLayout;
}

@freezed
class Workstation with _$Workstation {
  const factory Workstation({
    required String id,
    required String agentId,        // 智能体ID
    required Position position,     // 位置坐标
    required Size size,             // 工位大小
    String? customName,             // 自定义名称
  }) = _Workstation;
}

@freezed
class Position with _$Position {
  const factory Position({
    required double x,
    required double y,
  }) = _Position;
}

enum LayoutType {
  grid,        // 网格布局
  freeform,    // 自由布局
  departmental,// 部门分区
  dynamic,     // 动态生成
}
```

### 4.3 布局持久化

**保存时机：** 应用关闭时自动保存当前布局状态
**恢复时机：** 应用启动时自动恢复上次布局
**多设备同步：** 通过 PuupeeSync 在多设备间同步布局配置

## 5. 任务系统

### 5.1 任务生命周期

**状态流转：**
```
待分配 → 执行中 → 协作中 → 已完成
```

### 5.2 任务分配策略

**智能分配算法：**
1. **技能匹配** — 分析任务所需技能，匹配具有相应技能的智能体
2. **负载均衡** — 考虑智能体当前工作负载，避免过载
3. **协作优化** — 优先分配给可以协作的智能体组合
4. **历史表现** — 参考智能体历史任务完成情况

### 5.3 任务数据模型

```dart
@freezed
class Task with _$Task {
  const factory Task({
    required String id,
    required String title,          // 任务标题
    required String description,    // 任务描述
    required TaskStatus status,     // 任务状态
    required TaskPriority priority, // 优先级
    required List<String> requiredSkills, // 所需技能
    String? assignedAgentId,        // 分配的智能体ID
    List<String>? collaboratingAgentIds, // 协作智能体IDs
    String? parentTaskId,           // 父任务ID
    List<String>? subtaskIds,       // 子任务IDs
    Map<String, dynamic>? input,    // 输入数据
    Map<String, dynamic>? output,   // 输出数据
    DateTime? dueDate,              // 截止日期
    DateTime? completedAt,          // 完成时间
  }) = _Task;
}

enum TaskStatus {
  pending,       // 待分配
  assigned,      // 已分配
  inProgress,    // 执行中
  collaborating, // 协作中
  completed,     // 已完成
  failed,        // 失败
}

enum TaskPriority {
  low,           // 低优先级
  medium,        // 中优先级
  high,          // 高优先级
  urgent,        // 紧急
}
```

## 6. UI 设计

### 6.1 视图模式

**办公室视图（默认）：**
- 平面布局，卡通角色
- 工位显示智能体状态
- 工作流动画效果
- 过道显示智能体移动

**工作台视图：**
- 网格布局，详细信息
- 工位显示详细状态
- 任务进度展示
- 技能标签显示

**任务面板：**
- 列表视图，任务管理
- 任务创建和分配
- 进度跟踪
- 结果查看

### 6.2 响应式设计

**桌面端 (≥700px)：**
- 完整的工作区布局
- 多面板并排显示
- 详细的工位信息
- 丰富的交互功能

**移动端 (<700px)：**
- 简化的列表视图
- 单面板显示
- 核心信息展示
- 触屏友好的交互

### 6.3 办公室平面布局

**布局特点：**
- 6个工位的卡通角色
- 过道设计，支持智能体移动
- 状态指示器（空闲、工作中、离线）
- 工作交接动画效果

**动画效果：**
- **工作交接** — 智能体完成任务后，卡通形象移动到下一个工位
- **信息传递** — 智能体传递信息时，卡通形象移动到目标工位
- **协作会议** — 多个智能体协作时，卡通形象移动到会议室区域

## 7. 数据同步

### 7.1 同步架构

复用现有的 PuupeeSync 同步架构，支持：
- 本地数据存储
- 多设备同步
- 离线优先
- 冲突解决

### 7.2 同步内容

- 智能体配置
- 布局状态
- 任务数据
- 工作历史

## 8. 实现计划

### 8.1 第一阶段：基础框架

1. 创建 opc 子应用目录结构
2. 配置环境和依赖
3. 实现基础路由
4. 创建智能体数据模型

### 8.2 第二阶段：核心功能

1. 实现智能体管理
2. 实现布局引擎
3. 实现任务分配器
4. 实现办公室视图

### 8.3 第三阶段：高级功能

1. 实现工作流动画
2. 实现智能体通信
3. 实现数据同步
4. 优化性能和用户体验

## 9. 测试策略

### 9.1 单元测试

- 智能体状态管理测试
- 布局引擎测试
- 任务分配器测试

### 9.2 集成测试

- 智能体协作测试
- 数据同步测试
- 响应式布局测试

### 9.3 端到端测试

- 完整工作流测试
- 用户交互测试
- 性能测试

## 10. 风险和缓解

| 风险 | 影响 | 缓解措施 |
|------|------|----------|
| AI 服务不稳定 | 高 | 实现重试机制，支持离线模式 |
| 布局算法复杂 | 中 | 使用成熟的布局库，提供手动调整选项 |
| 数据同步冲突 | 中 | 使用现有的冲突解决机制 |
| 性能问题 | 中 | 优化渲染，使用懒加载 |

## 11. 成功标准

- [ ] 智能体可以正常工作和协作
- [ ] 布局可以动态生成和持久化
- [ ] 任务可以自动分配和跟踪
- [ ] 办公室视图可以正常显示和动画
- [ ] 数据可以多设备同步
- [ ] 响应式设计正常工作

## 12. 附录

### 12.1 术语表

- **智能体 (Agent)** — 具有特定技能和职责的 AI 角色
- **工位 (Workstation)** — 智能体在办公室中的位置
- **布局 (Layout)** — 办公室中工位的排列方式
- **任务 (Task)** — 需要智能体完成的工作
- **协作 (Collaboration)** — 多个智能体共同完成任务

### 12.2 参考资料

- PuupeeSync 同步架构文档
- Riverpod 状态管理文档
- shadcn_flutter UI 组件文档