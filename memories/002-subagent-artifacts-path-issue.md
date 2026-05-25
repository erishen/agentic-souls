# Memory: sub_agent artifacts 路径问题

## 问题 ID
`MEM-002`

## 发现时间
2026-05-17

## 问题描述

### 现象
在执行 Campaign 时，sub_agent (Specialist) 将 artifacts 文件放到了错误的位置：
- **错误位置**: `InvestKit/artifacts/phase-2/`, `InvestKit/artifacts/phase-3/`
- **正确位置**: `InvestKit/campaigns/2026-05-17-project-improvements/artifacts/phase-2/`, `phase-3/`

### 影响
1. artifacts 文件分散在不同位置
2. Campaign 目录结构不完整
3. 难以追溯执行过程

### 根因分析
1. sub_agent 没有正确理解相对路径
2. 任务描述中使用了相对路径 `artifacts/phase-2/`
3. sub_agent 默认使用项目根目录作为基准

## 解决方案

### 立即修复
手动移动文件到正确位置：
```bash
mv InvestKit/artifacts/phase-2 InvestKit/campaigns/xxx/artifacts/
mv InvestKit/artifacts/phase-3 InvestKit/campaigns/xxx/artifacts/
```

### 任务描述改进
在给 sub_agent 的任务描述中**必须使用绝对路径**：

```markdown
## 执行要求

1. 创建 `/Users/erishen/Workspace/CNB/individular-invest/InvestKit/campaigns/2026-05-17-project-improvements/artifacts/phase-2/` 目录
2. 将执行日志写入 `/Users/erishen/Workspace/CNB/individular-invest/InvestKit/campaigns/2026-05-17-project-improvements/artifacts/phase-2/execution.log`
```

### 文档改进
在 `souls/specialist.md` 中添加规则：

```yaml
规则_5:
  描述: 使用绝对路径创建 artifacts 文件
  原因: sub_agent 独立进程执行，相对路径可能解析错误
  详细: |
    在任务描述中必须明确指定：
    - artifacts 目录的绝对路径
    - 输出文件的绝对路径
    
    示例：
    - 正确: `/Users/xxx/InvestKit/campaigns/xxx/artifacts/phase-1/execution.log`
    - 错误: `artifacts/phase-1/execution.log`
```

## 验证方法

1. 检查 Campaign 目录结构是否完整
2. 检查 artifacts 是否在正确的 `campaigns/xxx/artifacts/` 下
3. 检查所有 Phase 的日志文件是否存在

## 相关文件

- `agentic-souls/souls/specialist.md` - 需要更新
- `InvestKit/campaigns/2026-05-17-project-improvements/` - 已修复

## 教训总结

1. **sub_agent 需要绝对路径** - 独立进程执行时相对路径可能解析错误
2. **任务描述要明确** - 不要假设 sub_agent 知道当前工作目录
3. **验证目录结构** - 每个 Phase 完成后检查文件位置

## 后续行动

- [x] 更新 `souls/specialist.md` 添加绝对路径规则 (2026-05-17 完成)
- [x] 更新 `docs/campaign.md` 说明 artifacts 路径要求 (2026-05-17 完成)
- [x] 在 Planner 任务描述模板中添加路径规范 (2026-05-17 完成，见 planner.md 规则_7)
