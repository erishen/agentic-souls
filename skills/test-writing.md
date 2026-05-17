# Test Writing Skill

> 测试编写技能

## Skill 定义

```yaml
name: test-writing
description: 测试编写技能
version: 1.0.0
category: testing
```

## 能力范围

```yaml
capabilities:
  - 编写单元测试
  - 编写集成测试
  - 编写 E2E 测试
  - 测试覆盖率分析
  
limitations:
  - 不编写生产代码
  - 不修改业务逻辑
```

## 测试规范

### 测试命名

```yaml
naming:
  file: test_{module}.py
  class: Test{Feature}
  method: test_{scenario}_{expected_result}
  
examples:
  - test_user.py
  - TestLogin
  - test_login_with_valid_credentials_returns_token
```

### 测试结构

```python
class TestFeature:
    """测试 FeatureX"""
    
    def test_scenario_1(self):
        """测试场景1: 正常情况"""
        # Arrange
        input_data = ...
        expected = ...
        
        # Act
        result = function_under_test(input_data)
        
        # Assert
        assert result == expected
    
    def test_scenario_2(self):
        """测试场景2: 边界情况"""
        ...
    
    def test_scenario_3(self):
        """测试场景3: 异常情况"""
        with pytest.raises(ExpectedException):
            function_under_test(invalid_input)
```

### 覆盖率要求

```yaml
coverage:
  statements: ">= 80%"
  branches: ">= 70%"
  functions: ">= 90%"
  
critical_paths:
  - 认证流程: 100%
  - 支付流程: 100%
  - 数据处理: 100%
```

## 测试类型

### 单元测试

```yaml
scope: 单个函数/方法
mock: 允许使用 mock
dependency: 隔离依赖
execution: 快速
```

### 集成测试

```yaml
scope: 多个模块协作
mock: 最小化 mock
dependency: 真实依赖
execution: 中等
```

### E2E 测试

```yaml
scope: 完整流程
mock: 不使用 mock
dependency: 真实环境
execution: 较慢
```

## 执行模板

### 接收任务

```markdown
## 测试任务接收

### 被测模块
- 模块: [模块名]
- 文件: [文件路径]
- 功能: [功能描述]

### 测试范围
- [ ] 正常流程
- [ ] 边界情况
- [ ] 异常处理

### 覆盖率目标
- 目标: [N]%
```

### 汇报结果

```markdown
## 测试任务汇报

### 完成内容
- [x] 编写了 [N] 个测试用例
- [x] 覆盖了 [N] 个场景
- [x] 发现了 [N] 个问题

### 测试结果
- 通过: [N]
- 失败: [N]
- 跳过: [N]

### 覆盖率
- 语句: [N]%
- 分支: [N]%
- 函数: [N]%

### 产物
- tests/test_xxx.py

### 发现的问题
- [问题1]
- [问题2]
```

## 示例

### 输入

```text
任务: 为 calculate_irr 函数编写测试
文件: src/finance/irr.py
覆盖率目标: 90%
```

### 输出

```python
# tests/test_irr.py
import pytest
from finance.irr import calculate_irr


class TestCalculateIRR:
    """测试 IRR 计算函数"""
    
    def test_normal_case_returns_valid_irr(self):
        """测试正常情况返回有效 IRR"""
        cashflows = [-100, 30, 30, 30, 30]
        irr = calculate_irr(cashflows)
        assert irr is not None
        assert 0 < irr < 1
    
    def test_simple_cashflows(self):
        """测试简单现金流"""
        cashflows = [-100, 110]
        irr = calculate_irr(cashflows)
        assert abs(irr - 0.1) < 0.01
    
    def test_empty_cashflows_raises_error(self):
        """测试空现金流抛出异常"""
        with pytest.raises(ValueError, match="现金流不能为空"):
            calculate_irr([])
    
    def test_all_positive_raises_error(self):
        """测试全正现金流抛出异常"""
        with pytest.raises(ValueError, match="现金流必须包含正负值"):
            calculate_irr([100, 50, 50])
    
    def test_all_negative_raises_error(self):
        """测试全负现金流抛出异常"""
        with pytest.raises(ValueError, match="现金流必须包含正负值"):
            calculate_irr([-100, -50, -50])
    
    def test_custom_guess(self):
        """测试自定义初始猜测值"""
        cashflows = [-1000, 300, 300, 300, 300]
        irr = calculate_irr(cashflows, guess=0.05)
        assert irr is not None
    
    def test_non_converging_returns_none(self):
        """测试不收敛情况返回 None"""
        cashflows = [-100, 1000, -2000, 3000]
        irr = calculate_irr(cashflows, max_iterations=10)
        # 可能不收敛
        assert irr is None or isinstance(irr, float)
```

## 常用命令

```bash
# 运行单个测试文件
pytest tests/test_xxx.py -v

# 运行特定测试
pytest tests/test_xxx.py::TestClass::test_method -v

# 运行并显示覆盖率
pytest tests/ --cov=src/ --cov-report=html

# 运行标记的测试
pytest -m "not slow"

# 并行运行
pytest -n auto
```
