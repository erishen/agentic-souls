# Dependency Upgrade Workflow

> 依赖升级工作流 — 兼容性优先，步步验证

## 工作流概述

```yaml
name: dependency-upgrade
description: 依赖升级的标准工作流，确保升级不破坏现有功能
version: 1.0.0
author: InvestKit Team
core_principle: 锁定范围，逐步升级，全量验证
```

## 触发条件

```yaml
triggers:
  - user_request: "升级依赖"
  - user_request: "更新版本"
  - security_alert: "安全漏洞告警"
  - deprecation_warning: "依赖即将废弃"
  - scheduled: "定期依赖更新"
```

## 工作流步骤

### Phase 1: 升级规划 (Planner)

```yaml
step: upgrade_planning
role: planner
actions:
  - 确认升级目标和范围
  - 分析当前依赖版本和目标版本
  - 识别破坏性变更（Breaking Changes）
  - 评估升级风险和影响面
  - 确定升级顺序（先核心后外围）
output:
  - 升级计划
  - 风险评估
  - 升级顺序
```

### Phase 2: 锁定与备份 (Specialist)

```yaml
step: lock_and_backup
role: specialist
type: code
actions:
  - 记录当前依赖锁定文件（uv.lock / package-lock.json）
  - 确认当前测试全部通过（建立基线）
  - 创建升级分支
output:
  - 依赖锁定文件备份
  - 基线测试报告
  - 升级分支
```

### Phase 3: 逐步升级 (Specialist)

```yaml
step: incremental_upgrade
role: specialist
type: code
actions:
  - 按计划顺序逐个升级依赖
  - 每次升级后运行测试
  - 处理 Breaking Changes（修改适配代码）
  - 记录每个依赖的升级结果
output:
  - 升级后的依赖文件
  - 适配代码修改
  - 升级日志
```

### Phase 4: 全量验证 (Specialist)

```yaml
step: full_verification
role: specialist
type: test
actions:
  - 运行全量测试套件
  - 检查依赖冲突
  - 验证构建产物
  - 安全漏洞扫描
  - 性能基准对比
output:
  - 测试报告
  - 依赖冲突报告
  - 安全扫描报告
  - 性能对比报告
```

### Phase 5: 验证 (Evaluator)

```yaml
step: verification
role: evaluator
actions:
  - 审核升级日志
  - 确认所有测试通过
  - 确认无安全漏洞
  - 确认性能未退化
  - 评估回滚风险
output:
  - 验证报告
  - 判决结果
```

## 工作流图

```
Upgrade Request
     │
     ▼
┌─────────────────┐
│ Phase 1         │
│ 升级规划        │ ◄── Planner
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Phase 2         │
│ 锁定与备份      │ ◄── Specialist (code)
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Phase 3         │
│ 逐步升级        │ ◄── Specialist (code)
│ (逐个依赖)      │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Phase 4         │
│ 全量验证        │ ◄── Specialist (test)
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
  strategy: lockfile_restore
  steps:
    - 恢复备份的依赖锁定文件
    - 执行依赖同步（uv sync / npm install）
    - 运行全量测试确认回滚成功
    - 通知 Planner 回滚结果

  trigger_conditions:
    - 测试大面积失败且无法快速修复
    - 存在无法解决的依赖冲突
    - 安全漏洞扫描发现新的高危漏洞
    - 性能退化超过阈值（>10%）
    - Evaluator 判决 FAIL

  prevention:
    - 升级前必须备份锁定文件
    - 逐个依赖升级，避免批量变更
    - 每步升级后立即运行测试
    - 保留升级分支以便回滚
```

## 验收标准模板

```yaml
acceptance_criteria:
  upgrade:
    - id: AC-1
      description: 目标依赖已升级到指定版本
      verification: 检查依赖文件版本号

    - id: AC-2
      description: 无依赖冲突
      verification: 依赖解析无警告

  compatibility:
    - id: AC-3
      description: 全量测试通过
      verification: 运行完整测试套件

    - id: AC-4
      description: Breaking Changes 已适配
      verification: 适配代码审查

  security:
    - id: AC-5
      description: 无已知高危安全漏洞
      verification: 安全扫描报告

  performance:
    - id: AC-6
      description: 性能未退化（不超过 10%）
      verification: 性能基准对比

  quality:
    - id: AC-7
      description: Lint 通过
      verification: ruff check / eslint
```

## 示例执行

```text
User: 升级 FastAPI 到最新版本，当前有安全漏洞

[Phase 1 - Planner]
升级规划:
  - 目标: FastAPI 0.100.x → 0.115.x
  - Breaking Changes: Pydantic v1 → v2 迁移
  - 风险: Pydantic 模型语法变更，影响面大
  - 顺序: 先升级 Pydantic v2 + 适配 → 再升级 FastAPI

[Phase 2 - Specialist]
锁定与备份:
  - 备份: uv.lock → uv.lock.bak
  - 基线测试: 156 passed ✅
  - 分支: feat/upgrade-fastapi

[Phase 3 - Specialist]
逐步升级:
  - Step 1: 升级 pydantic 到 v2 → 适配模型语法 → 测试 ✅
  - Step 2: 升级 fastapi 到 0.115.x → 适配路由变更 → 测试 ✅
  - Step 3: 升级 uvicorn 到最新兼容版本 → 测试 ✅

[Phase 4 - Specialist]
全量验证:
  - 测试: 156 passed ✅
  - 依赖冲突: 无 ✅
  - 安全扫描: 无高危漏洞 ✅
  - 性能: 响应时间与基线持平 ✅

[Phase 5 - Evaluator]
验证:
  - AC-1: ✅ FastAPI 已升级到 0.115.x
  - AC-2: ✅ 无依赖冲突
  - AC-3: ✅ 全量测试通过
  - AC-4: ✅ Pydantic v2 适配完成
  - AC-5: ✅ 无安全漏洞
  - AC-6: ✅ 性能持平
  - AC-7: ✅ Lint 通过
判决: PASS

[交付]
依赖升级完成: FastAPI 已升级到 0.115.x，安全漏洞已修复
```
