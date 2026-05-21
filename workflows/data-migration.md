# Data Migration Workflow

> 数据迁移工作流 — 安全第一，步步验证

## 工作流概述

```yaml
name: data-migration
description: 数据迁移的标准工作流，确保数据完整性和可回滚
version: 1.0.0
author: InvestKit Team
core_principle: 备份先行，步步验证，随时可回滚
```

## 触发条件

```yaml
triggers:
  - user_request: "数据迁移"
  - user_request: "数据库迁移"
  - schema_change: "表结构变更"
  - data_transform: "数据格式转换"
  - db_upgrade: "数据库升级"
```

## 工作流步骤

### Phase 1: 迁移规划 (Planner)

```yaml
step: migration_planning
role: planner
actions:
  - 确认迁移目标和范围
  - 分析源数据和目标结构
  - 评估数据量和迁移时间
  - 识别风险点和依赖关系
  - 制定迁移计划和时间窗口
output:
  - 迁移计划
  - 风险评估
  - 时间窗口
```

### Phase 2: 备份与验证 (Specialist)

```yaml
step: backup_and_verify
role: specialist
type: database
actions:
  - 执行完整数据库备份
  - 验证备份完整性
  - 记录备份位置和校验值
  - 在备份副本上验证迁移脚本
output:
  - 备份文件
  - 备份校验值
  - 备份验证报告
```

### Phase 3: 迁移执行 (Specialist)

```yaml
step: migration_execution
role: specialist
type: database
actions:
  - 执行迁移脚本
  - 监控迁移进度
  - 记录迁移日志
  - 处理迁移中的异常
output:
  - 迁移日志
  - 迁移执行报告
```

### Phase 4: 迁移验证 (Specialist)

```yaml
step: migration_verification
role: specialist
type: test
actions:
  - 数据完整性验证（行数、校验和）
  - 数据一致性验证（关键业务数据抽检）
  - Schema 验证（表结构符合预期）
  - 应用功能验证（读写正常）
  - 性能验证（查询性能可接受）
output:
  - 数据完整性报告
  - 数据一致性报告
  - Schema 验证报告
  - 功能验证报告
```

### Phase 5: 验证 (Evaluator)

```yaml
step: verification
role: evaluator
actions:
  - 审核所有验证报告
  - 确认数据完整性
  - 确认业务功能正常
  - 评估是否需要回滚
output:
  - 验证报告
  - 判决结果
```

## 工作流图

```
Migration Request
     │
     ▼
┌─────────────────┐
│ Phase 1         │
│ 迁移规划        │ ◄── Planner
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Phase 2         │
│ 备份与验证      │ ◄── Specialist (database)
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Phase 3         │
│ 迁移执行        │ ◄── Specialist (database)
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Phase 4         │
│ 迁移验证        │ ◄── Specialist (test)
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
  交付      回滚
```

## 回滚策略

```yaml
rollback:
  strategy: backup_restore
  steps:
    - 立即停止应用对数据库的写入
    - 确认备份文件位置和校验值
    - 执行备份恢复
    - 验证恢复后的数据完整性
    - 恢复应用连接
    - 通知 Planner 回滚结果

  trigger_conditions:
    - 数据完整性验证失败
    - 数据一致性验证失败
    - 业务功能验证失败
    - Evaluator 判决 FAIL
    - 迁移执行过程中出现不可恢复的错误

  time_constraint:
    - 回滚必须在迁移时间窗口内完成
    - 超过时间窗口需要升级处理

  prevention:
    - 迁移前必须完成备份
    - 备份必须验证完整性
    - 迁移脚本必须在备份副本上预演
    - 保留迁移前后的数据快照
```

## 验收标准模板

```yaml
acceptance_criteria:
  backup:
    - id: AC-1
      description: 完整备份已创建并验证
      verification: 备份文件存在且校验值匹配

    - id: AC-2
      description: 迁移脚本在备份副本上预演通过
      verification: 预演日志无错误

  integrity:
    - id: AC-3
      description: 数据行数一致
      verification: 源表行数 = 目标表行数

    - id: AC-4
      description: 关键字段校验和一致
      verification: 校验和对比

  consistency:
    - id: AC-5
      description: 关键业务数据抽检通过
      verification: 抽检 100 条核心数据

  functionality:
    - id: AC-6
      description: 应用读写功能正常
      verification: 运行集成测试

  schema:
    - id: AC-7
      description: 目标 Schema 符合预期
      verification: Schema 对比
```

## 示例执行

```text
User: 需要把 stock_daily 表从 MySQL 迁移到 PostgreSQL

[Phase 1 - Planner]
迁移规划:
  - 目标: stock_daily 表从 MySQL 迁移到 PostgreSQL
  - 数据量: 约 500 万行，预计 30 分钟
  - 风险: 数据类型差异、字符编码
  - 时间窗口: 凌晨 2:00 - 4:00
  - 计划: 备份 → 类型映射 → 批量迁移 → 验证

[Phase 2 - Specialist]
备份与验证:
  - MySQL 完整备份: backup_20260521_stock_daily.sql.gz
  - 备份大小: 1.2 GB
  - MD5: a1b2c3d4e5f6...
  - 备份验证: ✅ 完整
  - 预演: 在测试库执行迁移脚本 ✅ 通过

[Phase 3 - Specialist]
迁移执行:
  - 02:00 开始迁移
  - 类型映射: DATETIME → TIMESTAMP, TINYINT → SMALLINT
  - 批量大小: 10000 行/批
  - 02:25 迁移完成
  - 迁移日志: 5002341 行已迁移，0 错误

[Phase 4 - Specialist]
迁移验证:
  - 行数对比: MySQL 5002341 = PostgreSQL 5002341 ✅
  - 校验和: 关键字段校验和一致 ✅
  - 抽检: 100 条数据全部匹配 ✅
  - Schema: 符合预期 ✅
  - 功能: 读写测试通过 ✅

[Phase 5 - Evaluator]
验证:
  - AC-1: ✅ 备份完整
  - AC-2: ✅ 预演通过
  - AC-3: ✅ 行数一致
  - AC-4: ✅ 校验和一致
  - AC-5: ✅ 抽检通过
  - AC-6: ✅ 功能正常
  - AC-7: ✅ Schema 符合预期
判决: PASS

[交付]
数据迁移完成: stock_daily 表已成功从 MySQL 迁移到 PostgreSQL
```
