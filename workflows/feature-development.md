# Feature Development Workflow

> 新功能开发工作流

## 工作流概述

```yaml
name: feature-development
description: 新功能开发的标准工作流
version: 1.0.0
author: InvestKit Team
```

## 触发条件

```yaml
triggers:
  - user_request: "实现新功能"
  - user_request: "添加功能"
  - user_request: "开发功能"
```

## 角色配置

```yaml
roles:
  planner:
    soul: souls/planner.md
    responsibilities:
      - 需求分析
      - 任务分解
      - 委托执行
      - 验证交付
  
  evaluator:
    soul: souls/evaluator.md
    responsibilities:
      - 独立验证
      - 证据收集
      - 输出判决
  
  specialist:
    soul: souls/specialist.md
    types:
      - code: 代码实现
      - test: 测试编写
      - docs: 文档编写
```

## 工作流步骤

### Phase 1: 需求分析 (Planner)

```yaml
step: requirement_analysis
role: planner
actions:
  - 接收用户需求
  - 分析功能点
  - 识别技术约束
  - 定义验收标准
output:
  - 需求分析文档
  - 验收标准列表
```

### Phase 2: 任务分解 (Planner)

```yaml
step: task_decomposition
role: planner
actions:
  - 将需求分解为子任务
  - 确定任务依赖关系
  - 分配给合适的 Specialist
output:
  - 任务列表
  - 依赖关系图
  - 执行计划
```

### Phase 3: 并行执行 (Specialist)

```yaml
step: parallel_execution
role: specialist
actions:
  - 接收分配的子任务
  - 执行实现
  - 自检产物
  - 汇报结果
parallel: true
output:
  - 代码文件
  - 测试文件
  - 文档文件
```

### Phase 4: 结果收集 (Planner)

```yaml
step: result_collection
role: planner
actions:
  - 读取所有 Specialist 的输出
  - 检查是否满足验收标准
  - 决定是否需要修复
output:
  - 进度汇报
  - 问题列表
```

### Phase 5: 独立验证 (Evaluator)

```yaml
step: independent_verification
role: evaluator
actions:
  - 读取所有产物
  - 运行测试
  - 运行 Lint
  - 功能验证
  - 收集证据
output:
  - 验证报告
  - 判决结果
```

### Phase 6: 交付决策 (Planner)

```yaml
step: delivery_decision
role: planner
actions:
  - 接收 Evaluator 判决
  - PASS: 交付给用户
  - FAIL: 返回 Phase 3 修复
  - BLOCKED: 向用户报告
output:
  - 最终交付
  - 或修复任务
```

## 工作流图

```
┌──────────────────────────────────────────────────────────────────┐
│                   Feature Development Workflow                    │
└──────────────────────────────────────────────────────────────────┘

User Request
     │
     ▼
┌─────────────────┐
│ Phase 1         │
│ 需求分析        │ ◄── Planner
│ (Planner)       │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Phase 2         │
│ 任务分解        │ ◄── Planner
│ (Planner)       │
└────────┬────────┘
         │
         ▼
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│ Phase 3         │     │ Phase 3         │     │ Phase 3         │
│ 代码实现        │     │ 测试编写        │     │ 文档编写        │
│ (Specialist)    │     │ (Specialist)    │     │ (Specialist)    │
└────────┬────────┘     └────────┬────────┘     └────────┬────────┘
         │                       │                       │
         └───────────────────────┼───────────────────────┘
                                 │
                                 ▼
                    ┌─────────────────┐
                    │ Phase 4         │
                    │ 结果收集        │ ◄── Planner
                    │ (Planner)       │
                    └────────┬────────┘
                             │
                             ▼
                    ┌─────────────────┐
                    │ Phase 5         │
                    │ 独立验证        │ ◄── Evaluator
                    │ (Evaluator)     │
                    └────────┬────────┘
                             │
              ┌──────────────┼──────────────┐
              │              │              │
              ▼              ▼              ▼
           PASS           FAIL         BLOCKED
              │              │              │
              ▼              │              ▼
     ┌─────────────┐        │      ┌─────────────┐
     │ Phase 6     │        │      │ 报告阻塞    │
     │ 交付        │        │      └─────────────┘
     │ (Planner)   │        │
     └─────────────┘        │
                            │
                            └──────► 返回 Phase 3 修复
```

## 验收标准模板

```yaml
acceptance_criteria:
  functional:
    - id: AC-1
      description: 功能正常工作
      verification: 运行测试
      
    - id: AC-2
      description: 边界情况处理
      verification: 边界测试
      
  quality:
    - id: AC-3
      description: 测试覆盖率 >= 80%
      verification: pytest --cov
      
    - id: AC-4
      description: Lint 通过
      verification: ruff check
      
  documentation:
    - id: AC-5
      description: API 文档完整
      verification: 读取文档
```

## 示例执行

### 输入

```text
User: 帮我实现用户注册功能
```

### 执行过程

```text
[Phase 1 - Planner]
需求分析:
  - 功能: 用户注册
  - 输入: 用户名、密码、邮箱
  - 输出: 注册成功/失败
  - 验收标准:
    - AC-1: 注册 API 存在
    - AC-2: 密码加密存储
    - AC-3: 邮箱格式验证
    - AC-4: 用户名唯一性检查

[Phase 2 - Planner]
任务分解:
  - Task-1: 实现注册 API (Specialist-code)
  - Task-2: 实现密码加密 (Specialist-code)
  - Task-3: 编写单元测试 (Specialist-test)
  - Task-4: 编写 API 文档 (Specialist-docs)

[Phase 3 - Specialist]
Task-1 完成: src/api/register.py
Task-2 完成: src/utils/crypto.py
Task-3 完成: tests/test_register.py
Task-4 完成: docs/api/register.md

[Phase 4 - Planner]
结果收集:
  - 所有任务已完成
  - 产物已生成
  - 准备验证

[Phase 5 - Evaluator]
验证报告:
  - AC-1: ✅ PASS - 文件存在
  - AC-2: ✅ PASS - 使用 bcrypt
  - AC-3: ✅ PASS - 正则验证通过
  - AC-4: ✅ PASS - 唯一性检查存在
  - 测试: ✅ PASS - 5 passed
  - Lint: ✅ PASS - 无错误
判决: PASS

[Phase 6 - Planner]
交付: 用户注册功能已完成
```

## 配置选项

```yaml
options:
  max_retries: 3           # 最大重试次数
  timeout: 3600            # 超时时间(秒)
  parallel_tasks: true     # 允许并行任务
  auto_fix: true           # 自动修复失败项
  require_tests: true      # 必须有测试
  require_docs: true       # 必须有文档
```

## 错误处理

```yaml
error_handling:
  specialist_failure:
    action: 重试或分配给其他 Specialist
    max_retries: 2
    
  evaluator_fail:
    action: 返回修复阶段
    max_iterations: 3
    
  timeout:
    action: 向用户报告超时
    save_progress: true
```
