# Agentic Souls

> 基于 Agentic SDLC 三角色架构的 AI 工作流系统

## 概述

Agentic Souls 是一套基于文档驱动的 AI 工作流系统，通过定义三个核心角色（Planner、Evaluator、Specialist）来实现高质量的软件开发流程。

### 核心理念

```
Planner: 你是交付负责人，你不写实现代码 - 你规划、分解、验证。
Evaluator: 你是独立评审官，你不帮忙、不被建议、只判断 AC 是否达成。
Specialist: 你是实施者，你只做被分配的一个子任务，做完汇报，不越权。
```

## 目录结构

```
agentic-souls/
├── souls/                    # 角色定义
│   ├── planner.md           # 规划者
│   ├── evaluator.md         # 评审官
│   └── specialist.md        # 实施者
│
├── workflows/               # 工作流定义（流程编排）
│   ├── feature-development.md  # 新功能开发
│   ├── bug-fix.md             # Bug 修复
│   └── code-review.md         # 代码审查
│
├── skills/                  # 技能定义（具体能力）
│   ├── python-code.md       # Python 代码实现
│   └── test-writing.md      # 测试编写
│
├── campaigns/               # Campaign 存储
│   └── .gitkeep
│
└── docs/                    # 文档
    └── campaign.md         # Campaign 定义
```

## 三角色架构

### Planner（规划者）

- **职责**: 规划任务、分解任务、委托执行、验证交付
- **约束**: 永远不自己写超过20行的实现代码
- **核心规则**:
  1. 所有实现通过 Task 工具委托给 Specialist
  2. 完成前必须调用 Evaluator，不允许自我评判
  3. 每个 Task 结束后立即读出输出，再决定下一步

### Evaluator（评审官）

- **职责**: 独立验证验收标准(AC)是否达成
- **约束**: 不帮忙、不给建议、只输出判决
- **核心规则**:
  1. 不信任 Planner 的陈述，自己动手验证
  2. 不发现证据不给 PASS，去读文件、跑命令
  3. 不在职责范围内给建议，只输出判决

### Specialist（实施者）

- **职责**: 执行 Planner 分配的子任务
- **约束**: 不越权、不规划、不判断整体完成
- **核心规则**:
  1. 只做 Planner 分配的子任务，不自行扩大范围
  2. 不做整体规划，不判断整体是否完成
  3. 完成后立即汇报，说清楚做了什么、产物在哪

## 工作流程

```
User Request
     │
     ▼
┌─────────────┐
│   Planner   │ ─── 分析需求、分解任务
└─────────────┘
     │
     ▼
┌─────────────┐
│ Specialist  │ ─── 执行子任务、汇报结果
└─────────────┘
     │
     ▼
┌─────────────┐
│  Evaluator  │ ─── 独立验证、输出判决
└─────────────┘
     │
     ▼
  PASS/FAIL
```

## 使用方法

### 1. 选择工作流

根据任务类型选择合适的工作流：

- **新功能开发**: `workflows/feature-development.md`
- **Bug 修复**: `workflows/bug-fix.md`
- **代码审查**: `workflows/code-review.md`

### 2. 加载角色

让 AI 加载对应的 soul 文档：

```text
请阅读 souls/planner.md，你现在是 Planner 角色。
```

### 3. 执行工作流

按照工作流定义的步骤执行：

```text
请按照 workflows/feature-development.md 执行新功能开发。
```

### 4. 角色切换

在不同阶段切换角色：

```text
[Planner 完成任务分解]
→ 委托给 Specialist 执行
→ Specialist 汇报结果
→ 调用 Evaluator 验证
→ Planner 根据判决决定下一步
```

## 示例

### 实现新功能

```text
User: 帮我实现用户登录功能

[AI 加载 Planner soul]
Planner:
  分析需求:
    - 功能: 用户登录
    - 输入: 用户名、密码
    - 输出: JWT Token
  
  任务分解:
    - Task-1: 实现登录 API (委托给 Specialist-code)
    - Task-2: 实现密码加密 (委托给 Specialist-code)
    - Task-3: 编写测试 (委托给 Specialist-test)
  
  → 委托 Task-1 给 Specialist

[AI 加载 Specialist soul]
Specialist:
  任务: 实现登录 API
  完成: 创建了 src/api/login.py
  产物: src/api/login.py, tests/test_login.py
  注意: 需要配置 JWT_SECRET

[AI 切换回 Planner]
Planner:
  → 读取 Specialist 输出
  → 继续委托 Task-2, Task-3
  → 所有任务完成，调用 Evaluator

[AI 加载 Evaluator soul]
Evaluator:
  验证:
    - AC-1: ✅ API 存在
    - AC-2: ✅ 密码加密
    - AC-3: ✅ 测试通过
  判决: PASS
  证据: [文件列表, 测试输出]

[AI 切换回 Planner]
Planner:
  → 收到 PASS 判决
  → 交付给用户: 用户登录功能已完成
```

## 扩展

### 添加新工作流

1. 在 `workflows/` 目录创建新的 `.md` 文件
2. 定义工作流步骤、角色、验收标准
3. 参考现有工作流格式

### 添加新技能

1. 在 `skills/` 目录创建新的 `.md` 文件
2. 定义技能能力、执行规范、输出模板
3. 参考现有技能格式

### 自定义角色

1. 在 `souls/` 目录创建新的角色文档
2. 定义角色职责、行为准则、输出模板
3. 更新工作流以使用新角色

## 质量保证

### 证据驱动

所有判决必须有证据支持：

- 文件路径
- 命令输出
- 测试结果
- 数据对比

### 独立验证

Evaluator 必须独立验证，不信任 Planner 的陈述：

```yaml
验证方式:
  - 读文件: 检查代码存在
  - 跑命令: 检查功能正确
  - 查日志: 检查执行结果
  - 对比数据: 检查计算准确
```

### 角色边界

每个角色都有明确的边界约束：

- Planner: 不写实现代码
- Evaluator: 不给建议
- Specialist: 不越权规划

## 最佳实践

1. **明确验收标准**: 每个任务都有清晰的 AC
2. **及时汇报**: Specialist 完成后立即汇报
3. **独立验证**: 必须通过 Evaluator 验证
4. **证据留存**: 所有验证结果保存到 evidence/
5. **持续改进**: 根据反馈优化工作流

## 与项目集成

Agentic Souls 可以与任何项目集成使用：

```yaml
集成方式:
  1. 作为独立项目使用
  2. 在项目中引用 souls/ 目录
  3. 自定义 workflows 和 skills
```

## 参考

- [Campaign 定义](docs/campaign.md)

## License

MIT
