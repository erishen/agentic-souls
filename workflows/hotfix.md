# Hotfix Workflow

> 紧急修复工作流 — 快速止血，安全验证

## 工作流概述

```yaml
name: hotfix
description: 紧急修复的标准工作流，优先恢复服务，后续补充完善
version: 1.0.0
author: InvestKit Team
core_principle: 快速止血，最小变更，安全验证
```

## 触发条件

```yaml
triggers:
  - production_error: "生产环境报错"
  - service_down: "服务不可用"
  - data_corruption: "数据异常"
  - security_breach: "安全漏洞紧急修复"
  - user_request: "紧急修复"
  - user_request: "线上出问题了"
```

## 工作流步骤

### Phase 1: 紧急评估 (Planner)

```yaml
step: emergency_assessment
role: planner
actions:
  - 评估影响范围和严重程度
  - 确定是否为紧急修复（P0/P1）
  - 确定修复策略（止血 vs 根治）
  - 确定回滚还是热修复
  - 通知相关方
output:
  - 影响评估
  - 修复策略
  - 优先级判定
```

### Phase 2: 快速止血 (Specialist)

```yaml
step: quick_fix
role: specialist
type: code
actions:
  - 定位问题根因
  - 实现最小修复（仅修复问题，不做优化）
  - 确保修复不引入新问题
  - 编写回归测试
output:
  - 修复代码
  - 回归测试
  - 修复说明
```

### Phase 3: 快速验证 (Specialist)

```yaml
step: quick_verification
role: specialist
type: test
actions:
  - 运行回归测试
  - 运行核心功能冒烟测试
  - 验证修复有效
  - 确认无副作用
output:
  - 测试报告
  - 冒烟测试报告
```

### Phase 4: 验证 (Evaluator)

```yaml
step: verification
role: evaluator
actions:
  - 验证问题已修复
  - 验证无回归
  - 评估修复质量
  - 判断是否需要后续完善
output:
  - 验证报告
  - 判决结果
  - 后续完善建议（如有）
```

## 工作流图

```
Emergency Report
     │
     ▼
┌─────────────────┐
│ Phase 1         │
│ 紧急评估        │ ◄── Planner
└────────┬────────┘
         │
    ┌────┴────┐
    ▼         ▼
  回滚       热修复
    │         │
    │         ▼
    │   ┌─────────────────┐
    │   │ Phase 2         │
    │   │ 快速止血        │ ◄── Specialist (code)
    │   └────────┬────────┘
    │            │
    │            ▼
    │   ┌─────────────────┐
    │   │ Phase 3         │
    │   │ 快速验证        │ ◄── Specialist (test)
    │   └────────┬────────┘
    │            │
    ▼            ▼
┌─────────────────┐
│ Phase 4         │
│ 验证            │ ◄── Evaluator
└────────┬────────┘
         │
    ┌────┴────┐
    ▼         ▼
  PASS       FAIL
    │         │
    ▼         │
  交付        └──► 返回 Phase 2
```

## 回滚 vs 热修复决策

```yaml
decision:
  rollback:
    when:
      - 问题根因不明
      - 修复时间超过 30 分钟
      - 影响面持续扩大
      - 有可用的稳定版本
    steps:
      - 确认上一稳定版本
      - 执行版本回滚
      - 验证服务恢复
      - 后续离线排查

  hotfix:
    when:
      - 问题根因明确
      - 修复简单（< 30 分钟）
      - 回滚代价大于修复
      - 无可用稳定版本
    steps:
      - 从 main 创建 hotfix 分支
      - 实现最小修复
      - 快速验证
      - 合并到 main 并部署
```

## 验收标准模板

```yaml
acceptance_criteria:
  fix:
    - id: AC-1
      description: 问题已修复，服务恢复正常
      verification: 验证问题场景

    - id: AC-2
      description: 回归测试通过
      verification: 运行回归测试

  safety:
    - id: AC-3
      description: 无副作用
      verification: 冒烟测试通过

    - id: AC-4
      description: Lint 通过
      verification: ruff check

  follow_up:
    - id: AC-5
      description: 后续完善任务已创建（如需要）
      verification: 检查任务跟踪
```

## 示例执行

```text
User: 线上登录接口报 500 错误，用户无法登录！

[Phase 1 - Planner]
紧急评估:
  - 严重程度: P0（核心功能不可用）
  - 影响范围: 所有用户无法登录
  - 修复策略: 热修复（根因明确，修复简单）
  - 预计修复时间: 10 分钟

[Phase 2 - Specialist]
快速止血:
  - 根因: token 验证函数缺少 None 检查
  - 修复: 添加 None 值校验
  - 回归测试: tests/test_login_hotfix.py
  - 修复说明: 3 行代码变更

[Phase 3 - Specialist]
快速验证:
  - 回归测试: 8 passed ✅
  - 冒烟测试: 12/12 通过 ✅
  - 登录功能: 正常 ✅

[Phase 4 - Evaluator]
验证:
  - AC-1: ✅ 登录接口恢复正常
  - AC-2: ✅ 回归测试通过
  - AC-3: ✅ 无副作用
  - AC-4: ✅ Lint 通过
  - AC-5: 建议后续完善 token 验证的边界处理
判决: PASS

[交付]
紧急修复完成: 登录接口 500 错误已修复，服务恢复正常
后续建议: 完善 token 验证边界处理，增加更多防御性检查
```

## 紧急修复原则

```yaml
principles:
  - 最小变更: 只修复问题，不做额外优化
  - 快速验证: 核心功能验证优先，完整测试后补
  - 保留现场: 保留错误日志和现场信息
  - 及时沟通: 修复过程保持相关方同步
  - 后续完善: 紧急修复后安排后续完善任务
```
