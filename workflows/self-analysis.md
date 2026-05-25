# Self-Analysis Workflow

> 自我分析更新工作流 — Campaign 完成后自动反思、生成 Memory、闭环改进

## 工作流概述

```yaml
name: self-analysis
description: 自我分析更新工作流
version: 1.0.0
author: InvestKit Team
```

## 触发条件

```yaml
triggers:
  - campaign_completed: "Campaign 判决为 PASS 或 FAIL 后自动触发"
  - user_request: "自我分析"
  - user_request: "回顾总结"
  - periodic: "每5次 Campaign 后定期回顾"
```

## 设计原则

```yaml
核心原则:
  1. 闭环: 每个发现必须跟踪到完成
  2. 自动: Campaign 完成后自动触发，不依赖手动
  3. 可追溯: 所有改进都有 Memory 记录
  4. 最小化: 只记录真正有价值的经验，避免噪音
```

## 工作流步骤

### Phase 1: Campaign 回顾 (Evaluator)

```yaml
step: campaign_review
role: evaluator
trigger: Campaign verdict 生成后
actions:
  - 读取 verdict.md，提取判决结果
  - 读取 execution.md，提取执行过程
  - 读取 evidence.md，提取验证证据
  - 识别以下情况:
    - FAIL 的 AC: 哪些验收标准未通过，为什么
    - 执行中的 issues: 遇到了什么问题
    - 多次重试: 哪些任务需要反复修复
    - 阻塞项: 什么导致了 BLOCKED
    - 效率问题: 哪些步骤耗时过长
output:
  - campaign_review.md (回顾报告)
```

### Phase 2: 经验提取 (Evaluator)

```yaml
step: experience_extraction
role: evaluator
condition: Phase 1 发现值得记录的经验
actions:
  - 基于回顾结果，判断是否需要创建 Memory
  - Memory 创建条件 (满足任一):
    1. 发现新问题: 之前 memories 中未记录的问题
    2. 找到新方案: 比 memories 中已有方案更好的方法
    3. 工作流改进: 可以优化 soul/workflow/skill 的地方
    4. 重复犯错: 同一类型问题出现第2次以上
  - 不创建 Memory 的情况:
    1. 正常执行，无特殊问题
    2. 已有 Memory 覆盖了同类问题
    3. 一次性的、不具普遍性的问题
output:
  - 新的 memory 文件 (如有)
  - 更新已有 memory (如重复出现)
```

### Phase 3: 改进执行 (Planner)

```yaml
step: improvement_execution
role: planner
condition: Phase 2 生成了 Memory 且包含后续行动
actions:
  - 读取新创建/更新的 Memory
  - 提取后续行动列表
  - 判断行动类型:
    - 更新 soul 文档: 委托 Specialist 修改 souls/*.md
    - 更新 workflow 文档: 委托 Specialist 修改 workflows/*.md
    - 更新 skill 文档: 委托 Specialist 修改 skills/*.md
    - 更新 campaign.md: 委托 Specialist 修改 docs/campaign.md
  - 委托 Specialist 执行改进
  - 读取 Specialist 输出确认完成
output:
  - 更新后的 soul/workflow/skill 文档
  - Memory 中后续行动标记为 [x]
```

### Phase 4: 闭环验证 (Evaluator)

```yaml
step: closure_verification
role: evaluator
condition: Phase 3 完成后
actions:
  - 验证 Memory 中的后续行动是否已执行
  - 验证 soul/workflow/skill 文档是否已更新
  - 将 Memory 中的 - [ ] 改为 - [x]
  - 如果有未完成的行动，记录原因
output:
  - 闭环验证报告
  - 更新后的 memory 文件 (TODO 标记为完成)
```

## Memory 自动创建规则

### 创建时机

```yaml
自动创建:
  trigger: campaign_completed
  condition:
    - verdict 为 FAIL
    - execution 中有 issues
    - 同类问题出现第2次
    - 发现比现有 Memory 更好的方案

手动创建:
  trigger: user_request
  condition: 用户明确要求记录经验
```

### Memory 编号规则

```yaml
编号格式: NNN-简短标题.md
示例:
  - 001-campaign-workflow-issue.md
  - 002-subagent-artifacts-path-issue.md
  - 003-investment-data-accuracy.md
  - 004-self-analysis-workflow.md (本次新增)

规则:
  - 序号递增，不跳号
  - 标题用小写英文，短横线分隔
  - 标题概括问题本质，不超过5个词
```

### Memory 闭环规则

```yaml
后续行动跟踪:
  初始状态: - [ ] 行动描述
  完成状态: - [x] 行动描述 (YYYY-MM-DD 完成)

闭环条件:
  - soul/workflow/skill 文档已更新
  - 更新内容与后续行动一致
  - 下次 Campaign 执行时不再出现同类问题

未闭环处理:
  - 如果行动无法完成，记录原因
  - 标记为 - [~] 行动描述 (原因: xxx)
  - 下次 self-analysis 时重新评估
```

## 回顾报告模板

```markdown
## Campaign 回顾报告

### 基本信息
- Campaign ID: [CAMP-XXX]
- 任务类型: [feature/bugfix/refactor]
- 最终判决: [PASS/FAIL]
- 执行时间: [开始] → [结束]

### 执行质量
- 子任务总数: [N]
- 一次通过数: [N]
- 需要修复数: [N]
- 阻塞数: [N]

### 发现的问题
| # | 问题 | 类型 | 严重程度 | 是否新问题 |
|---|------|------|---------|-----------|
| 1 | [描述] | [执行/验证/流程] | [高/中/低] | [是/否] |

### 经验提取
- [ ] 需要创建新 Memory: [是/否]
- [ ] 需要更新已有 Memory: [是/否，哪个]
- [ ] 需要更新 soul/workflow/skill: [是/否，哪个]

### 改进行动
| # | 行动 | 目标文件 | 优先级 | 状态 |
|---|------|---------|--------|------|
| 1 | [描述] | [文件路径] | [高/中/低] | [ ] |

### 效率分析
- 总耗时: [N分钟]
- 最耗时步骤: [描述]
- 可优化步骤: [描述]
```

## 定期回顾机制

```yaml
定期回顾:
  触发: 每5次 Campaign 后
  执行者: Evaluator
  
  检查项:
    1. memories/ 目录中所有 Memory 的后续行动完成率
    2. 最近5次 Campaign 的 FAIL 率趋势
    3. 重复出现的问题类型
    4. soul/workflow/skill 文档是否需要更新
  
  输出:
    - periodic-review.md (定期回顾报告)
    - 更新未闭环的 Memory
    - 清理已过时的 Memory (标记为 archived)
```

## 与现有工作流的集成

### Campaign 流程扩展

```
原有流程:
  Task → Plan → Execution → Evidence → Verdict → 交付

扩展后流程:
  Task → Plan → Execution → Evidence → Verdict → Self-Analysis → 交付
                                                      │
                                                      ├─ Phase 1: 回顾
                                                      ├─ Phase 2: 提取经验
                                                      ├─ Phase 3: 执行改进
                                                      └─ Phase 4: 闭环验证
```

### Planner 职责扩展

```yaml
新增职责:
  - Campaign 完成后，触发 self-analysis workflow
  - 读取 Evaluator 的回顾报告
  - 委托 Specialist 执行改进行动
  - 确认 Memory 后续行动闭环
```

### Evaluator 职责扩展

```yaml
新增职责:
  - 在 verdict 后自动执行 Campaign 回顾
  - 提取经验并创建/更新 Memory
  - 验证改进行动的闭环
  - 定期回顾 memories 的有效性
```

## 注意事项

```yaml
重要约束:
  1. Self-Analysis 不应成为交付的阻塞项
     - 改进行动可以异步执行
     - 不影响 Campaign 的交付时间
  
  2. Memory 不应成为噪音
     - 只记录真正有价值的经验
     - 正常执行不创建 Memory
     - 重复问题才值得记录
  
  3. 改进要谨慎
     - soul/workflow/skill 的修改需要验证
     - 不能因为一次失败就大改流程
     - 同类问题出现2次以上才考虑修改文档
  
  4. 闭环要彻底
     - 后续行动要么完成，要么标记原因
     - 不允许长期停留在 - [ ] 状态
     - 超过3次 Campaign 未闭环的行动需要重新评估
```
