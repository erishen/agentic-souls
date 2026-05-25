# Memory: Self-Analysis Workflow 缺失

## 问题 ID
`MEM-004`

## 发现时间
2026-05-24

## 问题描述

### 现象
agentic-souls 项目没有自我分析更新的自动化流程：
1. Memories 中的后续行动长期停留在 `- [ ]` 状态，无人闭环
2. Campaign 完成后没有自动反思机制
3. 没有定期回顾 memories 有效性的机制
4. soul/workflow/skill 文档不会基于执行经验自动更新

### 影响
1. 经验记录了但没有落地改进
2. 同类问题可能重复出现
3. 工作流无法自我进化

### 根因分析
1. campaign.md 只定义了 Memory 触发条件，没有定义闭环流程
2. Evaluator 的职责只到 verdict 为止，没有延伸到反思
3. Planner 没有触发 self-analysis 的职责

## 解决方案

### 新增 self-analysis workflow
1. 创建 `workflows/self-analysis.md`，定义4阶段流程：
   - Phase 1: Campaign 回顾 (Evaluator)
   - Phase 2: 经验提取 (Evaluator)
   - Phase 3: 改进执行 (Planner)
   - Phase 4: 闭环验证 (Evaluator)

2. 更新 `souls/evaluator.md`：
   - 规则_5: verdict 后自动执行 Campaign 回顾
   - 规则_6: 验证 Memory 后续行动闭环

3. 更新 `souls/planner.md`：
   - 规则_7: Campaign 完成后触发 self-analysis workflow

4. 更新 `docs/campaign.md`：
   - 添加 Self-Analysis 自动触发规则
   - 添加 Memory 闭环规则（`[x]` `[~]` 状态）

5. 闭环现有3条 Memories 的 TODO

## 验证方法

1. 检查 workflows/self-analysis.md 是否存在且内容完整
2. 检查 evaluator.md 是否包含规则_5和规则_6
3. 检查 planner.md 是否包含规则_7
4. 检查 campaign.md 是否包含 Self-Analysis 自动触发规则
5. 检查3条 Memory 的后续行动是否已闭环

## 教训总结

1. **记录不等于改进** - Memory 只有闭环才算真正落地
2. **自动化优于手动** - 依赖人记得去检查 TODO 是不可靠的
3. **流程需要自我进化** - 工作流本身也需要被工作流改进

## 后续行动

- [x] 创建 workflows/self-analysis.md (2026-05-24 完成)
- [x] 更新 souls/evaluator.md 添加反思能力 (2026-05-24 完成)
- [x] 更新 souls/planner.md 添加 self-analysis 触发 (2026-05-24 完成)
- [x] 更新 docs/campaign.md 添加自动触发规则 (2026-05-24 完成)
- [x] 闭环 MEM-001、MEM-002、MEM-003 的后续行动 (2026-05-24 完成)
