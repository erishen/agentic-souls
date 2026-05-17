# Memory: Campaign 工作流问题记录

## 问题 ID
`MEM-001`

## 发现时间
2026-05-17

## 问题描述

### 现象
在执行 Campaign 时，Planner 一次性创建了所有文档（task.md, plan.md, execution.md, evidence.md, verdict.md），而不是按阶段逐步创建。

### 影响
1. 无法追踪执行过程
2. 无法展示阶段性日志
3. 违背了 agentic-souls 工作流设计原则

### 根因分析
1. 对工作流理解不够深入
2. campaign.md 文档没有明确说明阶段性创建的要求
3. 缺少对文档创建时机的约束

## 解决方案

### 立即修复
1. 删除多余的文档（execution.md, evidence.md, verdict.md）
2. 只保留 Planner 阶段产出的 task.md 和 plan.md

### 文档改进
1. 在 `docs/campaign.md` 中添加"阶段性工作流（重要）"章节
2. 明确说明文档创建时机和责任者
3. 添加错误示例和正确做法对比

### 流程约束
```
正确的文档创建流程：
Phase 1 开始 → Specialist 创建 execution.md
Phase 1 完成 → Evaluator 创建 evidence.md
Phase 2 开始 → Specialist 追加 execution.md
Phase 2 完成 → Evaluator 追加 evidence.md
...
所有 Phase 完成 → Evaluator 创建 verdict.md
```

## 验证方法

1. 检查 Campaign 目录，确认只有 task.md 和 plan.md
2. 后续 Phase 执行时，检查文档是否按阶段创建
3. 检查日志是否包含执行命令和输出

## 相关文件

- `agentic-souls/docs/campaign.md` - 已更新
- `InvestKit/campaigns/2026-05-17-unify-dependency-management/` - 已清理

## 教训总结

1. **工作流必须严格遵守** - 每个阶段有明确的职责和产出
2. **文档是过程的记录** - 不是预先填充的模板
3. **日志是验证的依据** - 必须展示实际执行过程

## 后续行动

- [ ] 在 planner.md 中添加"不要一次性创建所有文档"的规则
- [ ] 在 specialist.md 中添加"创建 execution.md"的职责说明
- [ ] 在 evaluator.md 中添加"创建 evidence.md 和 verdict.md"的职责说明
