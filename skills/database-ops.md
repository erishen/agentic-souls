# Database Ops Skill

> 数据库操作技能 — SQL 编写规范、迁移脚本、连接管理

## Skill 定义

```yaml
name: database-ops
description: 数据库操作技能，涵盖 SQL 编写、迁移脚本和连接管理
version: 1.0.0
category: infrastructure
```

## 能力范围

```yaml
capabilities:
  - 编写和优化 SQL 查询
  - 编写数据库迁移脚本
  - 管理 Schema 变更
  - 数据库连接池配置
  - 索引优化
  - 数据备份与恢复

limitations:
  - 不处理应用业务逻辑
  - 不涉及前端展示
  - 不处理非数据库基础设施
```

## 执行规范

### SQL 编写规范

```yaml
style:
  formatter: sqlfluff
  dialect: postgres / mysql

naming:
  table: snake_case, 复数形式 (如: stock_prices)
  column: snake_case (如: created_at)
  index: idx_{table}_{columns} (如: idx_stock_prices_date)
  constraint: fk_{table}_{ref_table} / uk_{table}_{columns}

rules:
  - 关键字大写 (SELECT, FROM, WHERE)
  - 表名和列名小写
  - 每个子句独占一行
  - 使用参数化查询，禁止字符串拼接
  - 查询必须指定字段，禁止 SELECT *
  - 大表查询必须带 LIMIT
  - JOIN 必须指定关联条件
  - WHERE 条件中禁止对列使用函数
```

### 迁移脚本规范

```yaml
migration:
  naming: "{timestamp}_{description}.sql"
  example: "20260521100000_add_stock_prices_table.sql"

  structure: |
    -- Up: 迁移内容
    CREATE TABLE stock_prices (
        id BIGSERIAL PRIMARY KEY,
        stock_code VARCHAR(10) NOT NULL,
        trade_date DATE NOT NULL,
        open_price DECIMAL(10,2),
        close_price DECIMAL(10,2),
        volume BIGINT,
        created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
    );

    CREATE INDEX idx_stock_prices_code_date
        ON stock_prices(stock_code, trade_date);

    -- Down: 回滚内容
    DROP TABLE IF EXISTS stock_prices;

  rules:
    - 每个迁移必须包含 Up 和 Down
    - 迁移脚本必须幂等
    - 数据迁移和 Schema 迁移分开
    - 大数据量迁移必须分批执行
    - 迁移前必须备份
    - 禁止在迁移中使用 DELETE 不带 WHERE
```

### 连接管理规范

```yaml
connection:
  pool:
    type: 连接池
    min_size: 5
    max_size: 20
    idle_timeout: 300s
    max_lifetime: 1800s

  transaction:
    - 保持事务短小
    - 禁止长事务
    - 只读操作使用只读连接
    - 事务内禁止远程调用

  error_handling:
    - 连接超时重试（最多 3 次）
    - 死锁检测和重试
    - 连接泄漏检测
    - 慢查询日志记录（>1s）
```

## 必须产出

```yaml
outputs:
  required:
    - SQL 文件 (.sql)
    - 迁移脚本 (含 Up/Down)

  optional:
    - 数据库文档
    - 性能优化报告
    - 索引分析报告
```

## 质量标准

```yaml
quality:
  sql_lint: "sqlfluff 无错误"
  migration: "Up/Down 成对"
  performance: "慢查询 < 1%"
  backup: "迁移前必须备份"
  idempotent: "迁移脚本幂等"
```

## 执行模板

### 接收任务

```markdown
## 任务接收

### 任务信息
- 任务: [任务描述]
- 数据库: [目标数据库]
- 验收标准: [AC列表]

### 执行计划
1. [步骤1]
2. [步骤2]
3. [步骤3]
```

### 汇报结果

```markdown
## 任务汇报

### 完成内容
- [x] 实现了 [功能1]
- [x] 编写了 [N] 个迁移脚本
- [x] 优化了 [查询]

### 产物
- migrations/xxx.sql
- docs/xxx.md

### 质量检查
- SQL Lint: ✅ 无错误
- 迁移测试: ✅ Up/Down 均通过
- 性能: ✅ 无慢查询

### 注意事项
- [需要关注的问题]
```

## 示例

### 输入

```text
任务: 为 stock_daily 表添加复合索引，优化按日期范围查询
验收标准:
  - 索引命名规范
  - 查询性能提升 50% 以上
  - 迁移脚本包含 Up/Down
```

### 输出

```sql
-- migrations/20260521100000_add_stock_daily_date_index.sql

-- Up
CREATE INDEX idx_stock_daily_code_date
    ON stock_daily(stock_code, trade_date);

-- Down
DROP INDEX IF EXISTS idx_stock_daily_code_date;
```

```sql
-- 验证查询性能
EXPLAIN ANALYZE
SELECT stock_code, trade_date, close_price
FROM stock_daily
WHERE stock_code = '000001'
  AND trade_date BETWEEN '2026-01-01' AND '2026-05-21'
LIMIT 100;
```

```text
性能对比:
  - 优化前: Seq Scan, 执行时间 850ms
  - 优化后: Index Scan, 执行时间 12ms
  - 提升: 98.6%
```

## 常用命令

```bash
# 运行迁移
alembic upgrade head

# 回滚迁移
alembic downgrade -1

# SQL Lint
sqlfluff lint migrations/

# 慢查询分析
EXPLAIN ANALYZE SELECT ...

# 连接池状态
SELECT * FROM pg_stat_activity;

# 索引使用分析
SELECT * FROM pg_stat_user_indexes;
```
