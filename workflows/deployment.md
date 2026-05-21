# Deployment Workflow

> 部署工作流 — 构建→测试→部署→健康检查→回滚

## 工作流概述

```yaml
name: deployment
description: 应用部署的标准工作流，确保安全上线
version: 1.0.0
author: InvestKit Team
core_principle: 自动化部署，全链路验证，快速回滚
```

## 触发条件

```yaml
triggers:
  - user_request: "部署"
  - user_request: "上线"
  - user_request: "发布"
  - merge_to_main: "合并到主分支"
  - scheduled: "定时部署"
```

## 工作流步骤

### Phase 1: 部署规划 (Planner)

```yaml
step: deployment_planning
role: planner
actions:
  - 确认部署目标和范围
  - 确认部署版本和变更内容
  - 评估部署风险和影响面
  - 确认部署时间窗口
  - 制定部署计划和回滚方案
output:
  - 部署计划
  - 变更清单
  - 回滚方案
```

### Phase 2: 构建与测试 (Specialist)

```yaml
step: build_and_test
role: specialist
type: devops
actions:
  - 拉取目标版本代码
  - 执行构建（编译、打包、镜像构建）
  - 运行单元测试
  - 运行集成测试
  - 代码质量检查（Lint、安全扫描）
  - 生成构建产物
output:
  - 构建产物（Docker 镜像 / 部署包）
  - 测试报告
  - 质量检查报告
  - 构建版本号
```

### Phase 3: 部署执行 (Specialist)

```yaml
step: deployment_execution
role: specialist
type: devops
actions:
  - 部署前环境检查
  - 执行部署（滚动更新 / 蓝绿部署）
  - 监控部署进度
  - 记录部署日志
output:
  - 部署日志
  - 部署执行报告
```

### Phase 4: 健康检查 (Specialist)

```yaml
step: health_check
role: specialist
type: test
actions:
  - 服务存活检查（/health 端点）
  - 核心功能冒烟测试
  - 性能指标检查（响应时间、错误率）
  - 日志异常检查
  - 依赖服务连通性检查
output:
  - 健康检查报告
  - 冒烟测试报告
  - 性能指标快照
```

### Phase 5: 验证 (Evaluator)

```yaml
step: verification
role: evaluator
actions:
  - 审核部署日志
  - 审核健康检查结果
  - 确认核心功能正常
  - 确认性能指标可接受
  - 评估是否需要回滚
output:
  - 验证报告
  - 判决结果
```

## 工作流图

```
Deployment Request
     │
     ▼
┌─────────────────┐
│ Phase 1         │
│ 部署规划        │ ◄── Planner
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Phase 2         │
│ 构建与测试      │ ◄── Specialist (devops)
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Phase 3         │
│ 部署执行        │ ◄── Specialist (devops)
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Phase 4         │
│ 健康检查        │ ◄── Specialist (test)
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
    ▼         ▼
  上线完成   执行回滚
```

## 回滚策略

```yaml
rollback:
  strategy: version_rollback
  steps:
    - 立即停止当前部署流程
    - 确认上一稳定版本号
    - 执行版本回滚（重新部署上一版本）
    - 等待回滚部署完成
    - 执行健康检查确认回滚成功
    - 通知 Planner 回滚结果

  trigger_conditions:
    - 健康检查失败
    - 冒烟测试不通过
    - 错误率超过阈值（>1%）
    - 响应时间超过阈值（>2x 基线）
    - Evaluator 判决 FAIL

  deployment_strategies:
    rolling_update:
      description: 滚动更新，逐步替换实例
      rollback: 回滚到上一镜像版本
      advantage: 零停机

    blue_green:
      description: 蓝绿部署，切换流量
      rollback: 切换流量回蓝环境
      advantage: 即时回滚

  prevention:
    - 部署前必须通过完整测试
    - 保留最近 3 个版本的构建产物
    - 部署前确认回滚方案
    - 部署过程保持监控
```

## 验收标准模板

```yaml
acceptance_criteria:
  build:
    - id: AC-1
      description: 构建成功，无错误
      verification: 构建日志无错误

    - id: AC-2
      description: 所有测试通过
      verification: 测试报告全绿

  deployment:
    - id: AC-3
      description: 部署成功，服务启动
      verification: 部署日志无错误

    - id: AC-4
      description: 健康检查通过
      verification: /health 返回 200

  functionality:
    - id: AC-5
      description: 核心功能冒烟测试通过
      verification: 冒烟测试报告

  performance:
    - id: AC-6
      description: 响应时间不超过基线 2 倍
      verification: 性能指标对比

    - id: AC-7
      description: 错误率不超过 1%
      verification: 监控面板
```

## 示例执行

```text
User: 部署 v2.3.0 到生产环境

[Phase 1 - Planner]
部署规划:
  - 版本: v2.3.0
  - 变更: 新增北向资金分析 API，优化数据查询性能
  - 风险: 数据库查询变更，可能影响响应时间
  - 时间窗口: 10:00 - 11:00
  - 策略: 滚动更新
  - 回滚方案: 重新部署 v2.2.1

[Phase 2 - Specialist]
构建与测试:
  - 构建: Docker 镜像 invest-api:v2.3.0 ✅
  - 单元测试: 156 passed ✅
  - 集成测试: 42 passed ✅
  - Lint: 无错误 ✅
  - 安全扫描: 无高危漏洞 ✅

[Phase 3 - Specialist]
部署执行:
  - 环境检查: 生产环境就绪 ✅
  - 滚动更新: 3/3 实例已更新 ✅
  - 部署耗时: 4 分 32 秒
  - 部署日志: 无错误

[Phase 4 - Specialist]
健康检查:
  - /health: 3/3 实例返回 200 ✅
  - 冒烟测试: 12/12 通过 ✅
  - 响应时间: 平均 120ms（基线 115ms）✅
  - 错误率: 0.02% ✅
  - 日志: 无异常错误 ✅

[Phase 5 - Evaluator]
验证:
  - AC-1: ✅ 构建成功
  - AC-2: ✅ 测试全通过
  - AC-3: ✅ 部署成功
  - AC-4: ✅ 健康检查通过
  - AC-5: ✅ 冒烟测试通过
  - AC-6: ✅ 响应时间正常
  - AC-7: ✅ 错误率正常
判决: PASS

[交付]
v2.3.0 已成功部署到生产环境
```
