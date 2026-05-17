# Code Review Workflow

> 代码审查工作流

## 工作流概述

```yaml
name: code-review
description: 代码审查工作流
version: 1.0.0
author: InvestKit Team
```

## 触发条件

```yaml
triggers:
  - pr_created: "Pull Request 创建"
  - user_request: "审查代码"
  - user_request: "Code Review"
```

## 工作流步骤

### Phase 1: 变更分析 (Planner)

```yaml
step: change_analysis
role: planner
actions:
  - 获取变更内容
  - 分析变更范围
  - 确定审查重点
output:
  - 变更摘要
  - 审查清单
```

### Phase 2: 代码审查 (Evaluator)

```yaml
step: code_review
role: evaluator
actions:
  - 检查代码风格
  - 检查逻辑正确性
  - 检查安全风险
  - 检查性能问题
  - 检查测试覆盖
output:
  - 审查报告
  - 问题列表
```

### Phase 3: 问题修复 (Specialist)

```yaml
step: issue_fix
role: specialist
condition: 有问题时执行
actions:
  - 修复审查发现的问题
  - 更新代码
output:
  - 修复代码
```

### Phase 4: 最终验证 (Evaluator)

```yaml
step: final_verification
role: evaluator
actions:
  - 验证问题已修复
  - 最终质量检查
output:
  - 最终判决
```

## 审查检查清单

```yaml
review_checklist:
  code_style:
    - 命名规范
    - 代码格式
    - 注释完整
    
  logic:
    - 逻辑正确
    - 边界处理
    - 异常处理
    
  security:
    - 输入验证
    - 权限检查
    - 敏感数据处理
    
  performance:
    - 算法效率
    - 资源使用
    - 潜在瓶颈
    
  testing:
    - 单元测试
    - 覆盖率
    - 测试质量
```

## 审查报告模板

```markdown
## Code Review Report

### 变更摘要
- 文件数: [N]
- 新增行: [N]
- 删除行: [N]
- 变更范围: [描述]

### 审查结果

#### 严重问题 (必须修复)
- [ ] [问题描述1]
- [ ] [问题描述2]

#### 建议改进 (可选)
- [ ] [建议1]
- [ ] [建议2]

#### 优点
- [优点1]
- [优点2]

### 判决
- [ ] APPROVE - 可以合并
- [ ] REQUEST_CHANGES - 需要修改
- [ ] COMMENT - 有建议但不阻塞
```
