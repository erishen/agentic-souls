# Python Code Skill

> Python 代码实现技能

## Skill 定义

```yaml
name: python-code
description: Python 代码实现技能
version: 1.0.0
category: implementation
```

## 能力范围

```yaml
capabilities:
  - 编写 Python 函数和类
  - 编写单元测试
  - 代码重构
  - 性能优化
  
limitations:
  - 不处理非 Python 代码
  - 不涉及部署配置
  - 不处理数据库迁移
```

## 执行规范

### 代码风格

```yaml
style:
  formatter: ruff
  type_checker: mypy
  docstring: google
  
naming:
  function: snake_case
  class: PascalCase
  constant: UPPER_SNAKE_CASE
  variable: snake_case
```

### 必须产出

```yaml
outputs:
  required:
    - 源代码文件 (.py)
    - 单元测试文件 (test_*.py)
    
  optional:
    - 类型存根文件 (.pyi)
    - 文档文件 (.md)
```

### 质量标准

```yaml
quality:
  test_coverage: ">= 80%"
  lint: "无错误"
  type_check: "无错误"
  docstring: "公共 API 必须有"
```

## 执行模板

### 接收任务

```markdown
## 任务接收

### 任务信息
- 任务: [任务描述]
- 文件: [目标文件]
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
- [x] 实现了 [功能2]
- [x] 编写了 [N] 个测试

### 产物
- src/xxx.py
- tests/test_xxx.py

### 质量检查
- 测试: ✅ [N] passed
- Lint: ✅ 无错误
- 覆盖率: [N]%

### 注意事项
- [需要关注的问题]
```

## 示例

### 输入

```text
任务: 实现一个计算 IRR 的函数
验收标准:
  - 函数签名: calculate_irr(cashflows: list[float]) -> float
  - 使用牛顿法计算
  - 处理边界情况
```

### 输出

```python
# src/finance/irr.py
from typing import Optional


def calculate_irr(cashflows: list[float], guess: float = 0.1, max_iterations: int = 100) -> Optional[float]:
    """
    使用牛顿法计算内部收益率(IRR)。
    
    Args:
        cashflows: 现金流列表，第一个为初始投资(负数)
        guess: 初始猜测值
        max_iterations: 最大迭代次数
        
    Returns:
        IRR 值，如果无法收敛则返回 None
        
    Raises:
        ValueError: 如果现金流为空或全为正/全为负
    """
    if not cashflows:
        raise ValueError("现金流不能为空")
    
    if all(cf >= 0 for cf in cashflows) or all(cf <= 0 for cf in cashflows):
        raise ValueError("现金流必须包含正负值")
    
    rate = guess
    
    for _ in range(max_iterations):
        npv = sum(cf / (1 + rate) ** i for i, cf in enumerate(cashflows))
        derivative = sum(-i * cf / (1 + rate) ** (i + 1) for i, cf in enumerate(cashflows))
        
        if abs(npv) < 1e-10:
            return rate
        
        if abs(derivative) < 1e-10:
            break
            
        rate = rate - npv / derivative
        
        if rate < -1:
            rate = -0.99
    
    return None if abs(npv) > 1e-6 else rate
```

```python
# tests/test_irr.py
import pytest
from finance.irr import calculate_irr


class TestCalculateIRR:
    def test_simple_case(self):
        cashflows = [-100, 30, 30, 30, 30]
        irr = calculate_irr(cashflows)
        assert irr is not None
        assert abs(irr - 0.077) < 0.01

    def test_empty_cashflows(self):
        with pytest.raises(ValueError):
            calculate_irr([])

    def test_all_positive(self):
        with pytest.raises(ValueError):
            calculate_irr([100, 50, 50])

    def test_all_negative(self):
        with pytest.raises(ValueError):
            calculate_irr([-100, -50, -50])
```

## 常用命令

```bash
# 运行测试
pytest tests/test_xxx.py -v

# 运行 Lint
ruff check src/

# 类型检查
mypy src/

# 测试覆盖率
pytest tests/ --cov=src/ --cov-report=term-missing
```
