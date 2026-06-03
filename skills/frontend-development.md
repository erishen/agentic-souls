# Frontend Development Skill

> 前端开发技能 — 组件设计、状态管理、样式方案

## Skill 定义

```yaml
name: frontend-development
description: 前端开发技能，涵盖 React/Next.js 组件开发、状态管理和样式方案
version: 1.0.0
category: implementation
```

## 能力范围

```yaml
capabilities:
  - React 组件开发（函数组件 / Hooks）
  - Next.js 应用开发（App Router）
  - 状态管理（Zustand / React Query）
  - 样式方案（Tailwind CSS / CSS Modules）
  - 前端测试（Jest / React Testing Library / Playwright）
  - 性能优化（代码分割 / 懒加载 / SSR）

limitations:
  - 不处理后端 API 逻辑
  - 不涉及数据库操作
  - 不处理基础设施部署
```

## 执行规范

### 组件设计规范

```yaml
component:
  style: 函数组件 + Hooks
  naming: PascalCase（如: StockChart, PriceTable）
  file_naming: PascalCase.tsx（如: StockChart.tsx）

  structure: |
    1. 类型定义（Props / State）
    2. Hooks（useState / useEffect / 自定义 Hooks）
    3. 事件处理函数
    4. 渲染逻辑
    5. 样式

  rules:
    - 单一职责：每个组件只做一件事
    - Props 向下流动，事件向上传递
    - 拆分超过 200 行的组件
    - 提取可复用逻辑为自定义 Hook
    - 使用 TypeScript 严格类型
```

### 状态管理规范

```yaml
state_management:
  local_state: useState / useReducer（组件内部状态）
  global_state: Zustand（跨组件共享状态）
  server_state: React Query / SWR（服务端数据缓存）

  rules:
    - 优先使用 local state
    - 服务端数据使用 React Query 管理
    - 全局状态最小化，只存必要数据
    - 避免状态冗余
```

### 样式规范

```yaml
styling:
  primary: Tailwind CSS
  fallback: CSS Modules
  naming: camelCase for CSS Modules

  rules:
    - 优先使用 Tailwind 工具类
    - 复杂样式提取为 CSS Modules
    - 响应式设计：mobile-first
    - 暗色模式支持
    - 避免内联 style 属性
```

### 测试规范

```yaml
testing:
  unit: Jest + React Testing Library
  e2e: Playwright
  coverage: ">= 80%"

  rules:
    - 测试用户行为，不测试实现细节
    - 使用 screen.getByRole 优先
    - Mock 最小化
    - 异步操作使用 waitFor
```

## 必须产出

```yaml
outputs:
  required:
    - React 组件文件 (.tsx)
    - 组件测试文件 (.test.tsx)
    - 类型定义文件 (.ts)

  optional:
    - Storybook stories (.stories.tsx)
    - CSS Modules 文件 (.module.css)
    - 自定义 Hooks 文件 (.ts)
```

## 质量标准

```yaml
quality:
  test_coverage: ">= 80%"
  type_check: "无 TypeScript 错误"
  lint: "eslint 无错误"
  accessibility: "无 a11y 严重问题"
  performance: "Lighthouse Performance >= 90"
```

## 执行模板

### 接收任务

```markdown
## 任务接收

### 任务信息
- 任务: [任务描述]
- 组件: [组件名称]
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
- [x] 实现了 [组件1]
- [x] 编写了 [N] 个测试
- [x] 通过了 a11y 检查

### 产物
- src/components/xxx.tsx
- src/components/xxx.test.tsx
- src/components/xxx.module.css

### 质量检查
- 测试: ✅ [N] passed
- TypeScript: ✅ 无错误
- ESLint: ✅ 无错误
- a11y: ✅ 无严重问题

### 注意事项
- [需要关注的问题]
```

## 示例

### 输入

```text
任务: 实现股票价格走势图组件
验收标准:
  - 使用 Recharts 展示折线图
  - 支持日期范围选择
  - 响应式布局
  - 加载状态和空状态处理
```

### 输出

```tsx
import { useState } from "react";
import {
  LineChart,
  Line,
  XAxis,
  YAxis,
  CartesianGrid,
  Tooltip,
  ResponsiveContainer,
} from "recharts";

interface PricePoint {
  date: string;
  close: number;
}

interface StockChartProps {
  data: PricePoint[];
  loading?: boolean;
  onDateRangeChange?: (from: string, to: string) => void;
}

export function StockChart({
  data,
  loading = false,
  onDateRangeChange,
}: StockChartProps) {
  if (loading) {
    return (
      <div className="flex h-64 items-center justify-center">
        <span className="text-gray-400">加载中...</span>
      </div>
    );
  }

  if (!data.length) {
    return (
      <div className="flex h-64 items-center justify-center">
        <span className="text-gray-400">暂无数据</span>
      </div>
    );
  }

  return (
    <div className="w-full">
      <ResponsiveContainer width="100%" height={300}>
        <LineChart data={data}>
          <CartesianGrid strokeDasharray="3 3" />
          <XAxis dataKey="date" fontSize={12} />
          <YAxis fontSize={12} />
          <Tooltip />
          <Line
            type="monotone"
            dataKey="close"
            stroke="#2563eb"
            strokeWidth={2}
            dot={false}
          />
        </LineChart>
      </ResponsiveContainer>
    </div>
  );
}
```

```tsx
import { render, screen } from "@testing-library/react";
import { StockChart } from "./StockChart";

describe("StockChart", () => {
  const mockData = [
    { date: "2026-01-01", close: 10.5 },
    { date: "2026-01-02", close: 11.0 },
    { date: "2026-01-03", close: 10.8 },
  ];

  it("renders chart with data", () => {
    render(<StockChart data={mockData} />);
    expect(screen.getByText("2026-01-01")).toBeInTheDocument();
  });

  it("shows loading state", () => {
    render(<StockChart data={[]} loading />);
    expect(screen.getByText("加载中...")).toBeInTheDocument();
  });

  it("shows empty state", () => {
    render(<StockChart data={[]} />);
    expect(screen.getByText("暂无数据")).toBeInTheDocument();
  });
});
```

## 常用命令

```bash
# 开发服务器
npm run dev

# 运行测试
npm test

# 类型检查
npx tsc --noEmit

# ESLint
npx eslint src/

# 构建生产版本
npm run build

# Lighthouse 审计
npx lighthouse http://localhost:3000 --output html
```
