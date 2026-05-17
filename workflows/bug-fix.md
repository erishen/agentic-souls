# Bug Fix Workflow

> Bug 修复工作流

## 工作流概述

```yaml
name: bug-fix
description: Bug 修复的标准工作流
version: 1.0.0
author: InvestKit Team
```

## 触发条件

```yaml
triggers:
  - user_request: "修复 bug"
  - user_request: "有个问题"
  - user_request: "报错了"
  - bug_report: "Bug 报告"
```

## 工作流步骤

### Phase 1: 问题分析 (Planner)

```yaml
step: problem_analysis
role: planner
actions:
  - 接收 Bug 描述
  - 分析问题原因
  - 确定影响范围
  - 定义修复目标
output:
  - 问题分析报告
  - 修复目标
```

### Phase 2: 复现验证 (Specialist)

```yaml
step: reproduction
role: specialist
type: test
actions:
  - 编写复现测试
  - 确认 Bug 存在
  - 记录复现步骤
output:
  - 复现测试
  - Bug 确认报告
```

### Phase 3: 修复实现 (Specialist)

```yaml
step: fix_implementation
role: specialist
type: code
actions:
  - 定位问题代码
  - 实现修复
  - 确保不引入新问题
output:
  - 修复代码
  - 修复说明
```

### Phase 4: 回归测试 (Specialist)

```yaml
step: regression_test
role: specialist
type: test
actions:
  - 运行所有相关测试
  - 确认 Bug 已修复
  - 确认无回归问题
output:
  - 测试报告
```

### Phase 5: 验证 (Evaluator)

```yaml
step: verification
role: evaluator
actions:
  - 验证 Bug 已修复
  - 验证无回归
  - 验证代码质量
output:
  - 验证报告
  - 判决结果
```

## 工作流图

```
Bug Report
     │
     ▼
┌─────────────────┐
│ Phase 1         │
│ 问题分析        │ ◄── Planner
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Phase 2         │
│ 复现验证        │ ◄── Specialist (test)
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Phase 3         │
│ 修复实现        │ ◄── Specialist (code)
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Phase 4         │
│ 回归测试        │ ◄── Specialist (test)
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Phase 5         │
│ 验证            │ ◄── Evaluator
└────────┬────────┘
         │
    ┌────┴────┐
    ▼         ▼
  PASS       FAIL
    │         │
    ▼         │
  交付        └──► 返回 Phase 3
```

## 验收标准模板

```yaml
acceptance_criteria:
  fix:
    - id: AC-1
      description: Bug 已修复
      verification: 运行复现测试
      
    - id: AC-2
      description: 复现测试通过
      verification: pytest test_bug_xxx.py
      
  regression:
    - id: AC-3
      description: 无回归问题
      verification: 运行所有测试
      
  quality:
    - id: AC-4
      description: Lint 通过
      verification: ruff check
```

## 示例执行

```text
User: 登录功能报错了，提示 "Invalid token"

[Phase 1 - Planner]
问题分析:
  - 现象: 登录后提示 Invalid token
  - 可能原因: Token 生成或验证问题
  - 影响范围: 登录功能
  - 修复目标: Token 正常工作

[Phase 2 - Specialist]
复现测试: tests/test_login_token.py
Bug 确认: Token 过期时间设置错误

[Phase 3 - Specialist]
修复: src/utils/token.py
修改: 将过期时间从 1 小时改为 24 小时

[Phase 4 - Specialist]
测试报告: 所有测试通过，无回归

[Phase 5 - Evaluator]
验证:
  - AC-1: ✅ Bug 已修复
  - AC-2: ✅ 复现测试通过
  - AC-3: ✅ 无回归
  - AC-4: ✅ Lint 通过
判决: PASS

[交付]
Bug 已修复: Token 过期时间问题已解决
```
