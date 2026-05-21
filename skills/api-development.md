# API Development Skill

> API 开发技能 — RESTful 设计、错误处理、认证授权

## Skill 定义

```yaml
name: api-development
description: API 开发技能，涵盖 RESTful 设计、错误处理和认证授权
version: 1.0.0
category: implementation
```

## 能力范围

```yaml
capabilities:
  - 设计和实现 RESTful API
  - 统一错误处理
  - 认证授权实现
  - API 版本管理
  - 请求参数验证
  - API 文档编写

limitations:
  - 不处理前端 UI 逻辑
  - 不涉及数据库底层优化
  - 不处理基础设施部署
```

## 执行规范

### RESTful 设计规范

```yaml
design:
  url_naming:
    - 使用名词复数: /api/v1/stocks
    - 嵌套资源: /api/v1/stocks/{code}/prices
    - 最多两层嵌套
    - 使用小写和连字符: /api/v1/stock-prices

  http_methods:
    GET: 获取资源，不修改状态
    POST: 创建资源
    PUT: 全量更新资源
    PATCH: 部分更新资源
    DELETE: 删除资源

  status_codes:
    200: 请求成功
    201: 创建成功
    204: 删除成功（无返回内容）
    400: 请求参数错误
    401: 未认证
    403: 无权限
    404: 资源不存在
    409: 资源冲突
    422: 参数验证失败
    429: 请求频率超限
    500: 服务器内部错误

  filtering:
    - 精确过滤: ?stock_code=000001
    - 范围过滤: ?date_from=2026-01-01&date_to=2026-05-21
    - 排序: ?sort=-date (降序) / ?sort=date (升序)
    - 分页: ?page=1&page_size=20
```

### 错误处理规范

```yaml
error_handling:
  response_format:
    type: object
    properties:
      error:
        type: object
        properties:
          code: "ERROR_CODE"
          message: "用户可读的错误描述"
          details: "详细错误信息（可选，开发环境）"

  error_codes:
    VALIDATION_ERROR: 参数验证失败
    AUTHENTICATION_ERROR: 认证失败
    AUTHORIZATION_ERROR: 权限不足
    RESOURCE_NOT_FOUND: 资源不存在
    RESOURCE_CONFLICT: 资源冲突
    RATE_LIMIT_EXCEEDED: 请求频率超限
    INTERNAL_ERROR: 服务器内部错误

  rules:
    - 永远不暴露内部实现细节
    - 永远不暴露堆栈信息给客户端
    - 5xx 错误记录完整日志
    - 4xx 错误返回明确的修复建议
    - 错误响应格式全局统一
```

### 认证授权规范

```yaml
authentication:
  method: JWT Bearer Token
  header: Authorization: Bearer <token>

  token_structure:
    header:
      alg: HS256
      typ: JWT
    payload:
      sub: 用户ID
      role: 用户角色
      iat: 签发时间
      exp: 过期时间

  token_rules:
    - Access Token 有效期: 2 小时
    - Refresh Token 有效期: 7 天
    - Token 存储在 HttpOnly Cookie 或 Authorization Header
    - 敏感操作需要二次验证

authorization:
  model: RBAC (基于角色的访问控制)
  roles:
    admin: 全部权限
    user: 读写自有资源
    viewer: 只读权限

  rules:
    - 默认拒绝，显式允许
    - 每个 API 端点必须声明所需权限
    - 资源级别权限检查（用户只能操作自己的资源）
    - 管理员操作需要审计日志
```

## 必须产出

```yaml
outputs:
  required:
    - API 实现代码
    - API 测试文件
    - API 文档 (OpenAPI/Swagger)

  optional:
    - 认证中间件
    - 请求/响应 Schema
    - 错误码文档
```

## 质量标准

```yaml
quality:
  test_coverage: ">= 80%"
  api_doc: "所有端点必须有文档"
  error_handling: "所有端点必须有错误处理"
  auth_check: "所有非公开端点必须有认证"
  input_validation: "所有输入必须验证"
  response_format: "响应格式全局统一"
```

## 执行模板

### 接收任务

```markdown
## 任务接收

### 任务信息
- 任务: [任务描述]
- API 路径: [端点路径]
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
- [x] 实现了 [API 端点]
- [x] 编写了 [N] 个测试
- [x] 编写了 API 文档

### 产物
- src/api/xxx.py
- tests/test_xxx.py
- docs/api/xxx.yaml

### 质量检查
- 测试: ✅ [N] passed
- 文档: ✅ 所有端点已文档化
- 认证: ✅ 已实现
- 错误处理: ✅ 已实现

### 注意事项
- [需要关注的问题]
```

## 示例

### 输入

```text
任务: 实现股票数据查询 API
验收标准:
  - GET /api/v1/stocks/{code}/prices 支持日期范围过滤
  - 支持分页
  - 需要 user 角色认证
  - 统一错误处理
```

### 输出

```python
from datetime import date
from fastapi import APIRouter, Depends, HTTPException, Query
from pydantic import BaseModel

router = APIRouter(prefix="/api/v1/stocks", tags=["stocks"])


class PriceResponse(BaseModel):
    stock_code: str
    trade_date: date
    open_price: float
    close_price: float
    high_price: float
    low_price: float
    volume: int


class PriceListResponse(BaseModel):
    items: list[PriceResponse]
    total: int
    page: int
    page_size: int


class ErrorResponse(BaseModel):
    error: dict


@router.get(
    "/{code}/prices",
    response_model=PriceListResponse,
    responses={404: {"model": ErrorResponse}, 401: {"model": ErrorResponse}},
)
async def get_stock_prices(
    code: str,
    date_from: date | None = Query(None, description="开始日期"),
    date_to: date | None = Query(None, description="结束日期"),
    page: int = Query(1, ge=1, description="页码"),
    page_size: int = Query(20, ge=1, le=100, description="每页数量"),
    current_user=Depends(get_current_user),
):
    prices = await fetch_prices(
        code=code,
        date_from=date_from,
        date_to=date_to,
        page=page,
        page_size=page_size,
    )
    if not prices and not await stock_exists(code):
        raise HTTPException(status_code=404, detail="RESOURCE_NOT_FOUND")
    return prices
```

```python
from fastapi.testclient import TestClient


class TestStockPricesAPI:
    def test_get_prices_success(self, client: TestClient, auth_headers):
        response = client.get(
            "/api/v1/stocks/000001/prices",
            params={"date_from": "2026-01-01", "date_to": "2026-05-21"},
            headers=auth_headers,
        )
        assert response.status_code == 200
        data = response.json()
        assert "items" in data
        assert "total" in data

    def test_get_prices_unauthorized(self, client: TestClient):
        response = client.get("/api/v1/stocks/000001/prices")
        assert response.status_code == 401

    def test_get_prices_not_found(self, client: TestClient, auth_headers):
        response = client.get(
            "/api/v1/stocks/999999/prices",
            headers=auth_headers,
        )
        assert response.status_code == 404

    def test_get_prices_pagination(self, client: TestClient, auth_headers):
        response = client.get(
            "/api/v1/stocks/000001/prices",
            params={"page": 1, "page_size": 10},
            headers=auth_headers,
        )
        assert response.status_code == 200
        assert len(response.json()["items"]) <= 10
```

## 常用命令

```bash
# 启动开发服务器
uvicorn src.main:app --reload

# 运行 API 测试
pytest tests/test_api/ -v

# 生成 OpenAPI 文档
python -m src.scripts.export_openapi

# API Lint
spectral lint docs/api/

# 运行所有测试
pytest tests/ --cov=src/api/ --cov-report=term-missing
```
