# Refactoring Workflow

> 重构工作流 — 核心原则：行为不变

## 工作流概述

```yaml
name: refactoring
description: 代码重构的标准工作流，确保行为不变
version: 1.0.0
author: InvestKit Team
core_principle: 行为不变 — 重构不改变外部可观测行为
```

## 触发条件

```yaml
triggers:
  - user_request: "重构"
  - user_request: "优化代码结构"
  - user_request: "代码太乱了"
  - code_review: "建议重构"
  - tech_debt: "技术债务清理"
```

## 工作流步骤

### Phase 1: 重构分析 (Planner)

```yaml
step: refactoring_analysis
role: planner
actions:
  - 确认重构目标
  - 分析现有代码结构
  - 识别重构范围和影响面
  - 确认行为基线（现有测试覆盖情况）
  - 制定重构计划
output:
  - 重构计划
  - 影响范围分析
  - 行为基线记录
```

### Phase 2: 基线建立 (Specialist)

```yaml
step: baseline_establishment
role: specialist
type: test
actions:
  - 确认现有测试覆盖
  - 补充缺失的对比测试
  - 建立性能基准
  - 记录当前行为快照
output:
  - 对比测试套件
  - 性能基准数据
  - 行为快照
```

### Phase 3: 重构实现 (Specialist)

```yaml
step: refactoring_implementation
role: specialist
type: code
actions:
  - 小步重构，每次只改一处
  - 每步重构后运行对比测试
  - 保持行为不变
  - 记录每步变更
output:
  - 重构后的代码
  - 变更记录
```

### Phase 4: 兼容性检查 (Specialist)

```yaml
step: compatibility_check
role: specialist
type: test
actions:
  - 运行全量对比测试
  - 性能基准对比
  - API 兼容性验证
  - 接口签名不变验证
output:
  - 对比测试报告
  - 性能对比报告
  - 兼容性报告
```

### Phase 5: 验证 (Evaluator)

```yaml
step: verification
role: evaluator
actions:
  - 验证行为不变
  - 验证性能未退化
  - 验证兼容性
  - 验证代码质量提升
  - 评估回滚风险
output:
  - 验证报告
  - 判决结果
```

## 工作流图

```
Refactoring Request
     │
     ▼
┌─────────────────┐
│ Phase 1         │
│ 重构分析        │ ◄── Planner
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Phase 2         │
│ 基线建立        │ ◄── Specialist (test)
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Phase 3         │
│ 重构实现        │ ◄── Specialist (code)
│ (小步迭代)      │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Phase 4         │
│ 兼容性检查      │ ◄── Specialist (test)
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
  交付        └──► 执行回滚策略
```

## 回滚策略

```yaml
rollback:
  strategy: git_revert
  steps:
    - 停止所有进行中的重构操作
    - 确认需要回滚的 commit 范围
    - 执行 git revert 回滚变更
    - 运行全量测试确认回滚成功
    - 通知 Planner 回滚结果

  trigger_conditions:
    - 对比测试失败且无法快速修复
    - 性能退化超过阈值（>10%）
    - 兼容性检查不通过
    - Evaluator 判决 FAIL

  prevention:
    - 小步提交，每步可独立回滚
    - 重构前确保 git 状态干净
    - 记录每步变更的 commit hash
```

## 验收标准模板

```yaml
acceptance_criteria:
  behavior:
    - id: AC-1
      description: 行为不变 — 所有对比测试通过
      verification: 运行对比测试套件

    - id: AC-2
      description: API 接口签名不变
      verification: 对比重构前后接口定义

  performance:
    - id: AC-3
      description: 性能未退化（不超过 10%）
      verification: 性能基准对比

  compatibility:
    - id: AC-4
      description: 向后兼容
      verification: 兼容性测试通过

  quality:
    - id: AC-5
      description: 代码质量提升
      verification: Lint 通过 + 复杂度降低

    - id: AC-6
      description: 测试覆盖不降低
      verification: 覆盖率报告
```

## 示例执行

```text
User: 重构 stock-data-fetch 模块，代码太乱了

[Phase 1 - Planner]
重构分析:
  - 目标: 重构 stock-data-fetch 模块，提升可读性和可维护性
  - 范围: src/data/stock_fetch.py 及相关文件
  - 影响面: 3 个模块依赖此文件
  - 行为基线: 现有测试覆盖 72%
  - 计划: 拆分大函数、提取公共逻辑、优化命名

[Phase 2 - Specialist]
基线建立:
  - 补充对比测试: tests/test_stock_fetch_behavior.py
  - 性能基准: fetch_daily_data 平均 1.2s
  - 行为快照: 记录 10 个典型输入的输出

[Phase 3 - Specialist]
重构实现:
  - Step 1: 提取 fetch_and_parse 函数 → 对比测试 ✅
  - Step 2: 拆分 fetch_daily_data 为多个子函数 → 对比测试 ✅
  - Step 3: 优化变量命名 → 对比测试 ✅
  - Step 4: 提取公共常量 → 对比测试 ✅

[Phase 4 - Specialist]
兼容性检查:
  - 对比测试: 全部通过 ✅
  - 性能对比: fetch_daily_data 平均 1.1s（提升 8%）✅
  - API 兼容性: 接口签名不变 ✅

[Phase 5 - Evaluator]
验证:
  - AC-1: ✅ 对比测试全部通过
  - AC-2: ✅ 接口签名不变
  - AC-3: ✅ 性能提升 8%
  - AC-4: ✅ 向后兼容
  - AC-5: ✅ 圈复杂度从 15 降到 6
  - AC-6: ✅ 测试覆盖从 72% 提升到 85%
判决: PASS

[交付]
重构完成: stock-data-fetch 模块可读性和可维护性显著提升
```
