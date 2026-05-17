# Campaign 定义

> Campaign 是一个完整的执行单元，包含从任务定义到最终判决的全过程。

## 概述

```yaml
Campaign:
  定义: 一个完整的任务执行单元
  组成: Task → Plan → Execution → Evidence → Verdict
  目的: 确保任务从定义到验证的完整追溯
```

## Campaign 结构

```yaml
campaign:
  id: CAMP-XXX
  name: [Campaign 名称]
  status: pending | in_progress | completed | failed
  
  task:           # 任务定义
    ...
    
  plan:           # 执行计划
    ...
    
  execution:      # 执行记录
    ...
    
  evidence:       # 验证证据
    ...
    
  verdict:        # 最终判决
    ...
```

---

## 1. Task（任务定义）

### 定义

```yaml
task:
  id: TASK-XXX
  type: feature | bugfix | refactor | docs | test
  priority: high | medium | low
  
  description:
    background: [背景说明]
    objective: [目标描述]
    scope: [范围边界]
    
  acceptance_criteria:
    - id: AC-1
      description: [验收标准描述]
      verification: [验证方法]
      
    - id: AC-2
      description: [验收标准描述]
      verification: [验证方法]
```

### 示例

```yaml
task:
  id: TASK-001
  type: feature
  priority: high
  
  description:
    background: 用户需要登录系统才能使用个性化功能
    objective: 实现用户登录功能
    scope: 
      - 登录 API
      - 密码加密
      - Token 生成
      - 不包含：注册、第三方登录
    
  acceptance_criteria:
    - id: AC-1
      description: POST /api/login 端点存在
      verification: 读取文件，确认路由定义
      
    - id: AC-2
      description: 密码使用 bcrypt 加密
      verification: 读取代码，确认使用 bcrypt 库
      
    - id: AC-3
      description: 返回有效的 JWT Token
      verification: 运行测试，验证 Token 格式
      
    - id: AC-4
      description: 测试覆盖率 >= 80%
      verification: pytest --cov
      
    - id: AC-5
      description: Lint 通过
      verification: ruff check
```

---

## 2. Plan（执行计划）

> ⚠️ **重要**: 每个 Phase 必须包含明确的验收标准 (AC)，Evaluator 将基于 AC 进行验证。

### 定义

```yaml
plan:
  created_by: Planner
  created_at: [时间戳]
  
  analysis:
    requirements: [需求分析]
    constraints: [约束条件]
    risks: [风险识别]
    
  phases:
    - phase_id: Phase-1
      name: [阶段名称]
      steps:
        - step: [步骤描述]
          action: [执行动作]
      acceptance_criteria:  # 必须包含 AC
        - id: AC-1.1
          description: [验收标准描述]
          verification: [验证方法]
          
  decomposition:
    - task_id: SUB-001
      name: [子任务名称]
      type: code | test | docs
      specialist: [Specialist 类型]
      dependencies: [依赖的任务]
      estimated_complexity: low | medium | high
      
    - task_id: SUB-002
      ...
      
  execution_order:
    parallel: [可并行执行的任务]
    sequential: [需顺序执行的任务]
    
  timeline:
    estimated: [预估时间]
    milestones: [里程碑]
```

### AC 标准格式

每个 Phase 的验收标准必须包含：

| 字段 | 说明 | 示例 |
|------|------|------|
| `id` | AC 唯一标识 | `AC-1.1` (Phase 1 的第 1 个 AC) |
| `description` | 验收标准描述 | `uv.lock` 文件存在 |
| `verification` | 验证方法 | `ls projects/lobster/uv.lock` |

### 示例

```yaml
plan:
  created_by: Planner
  created_at: 2024-01-15T10:00:00Z
  
  analysis:
    requirements:
      - 用户名密码登录
      - JWT Token 认证
      - 密码安全存储
    constraints:
      - 使用 Python FastAPI
      - 使用 bcrypt 加密
      - 使用 PyJWT 生成 Token
    risks:
      - 密码存储安全风险: 使用 bcrypt 缓解
      - Token 过期处理: 设置合理过期时间
      
  decomposition:
    - task_id: SUB-001
      name: 实现登录 API
      type: code
      specialist: code
      dependencies: []
      estimated_complexity: medium
      acceptance_criteria: [AC-1]
      
    - task_id: SUB-002
      name: 实现密码加密
      type: code
      specialist: code
      dependencies: []
      estimated_complexity: low
      acceptance_criteria: [AC-2]
      
    - task_id: SUB-003
      name: 编写单元测试
      type: test
      specialist: test
      dependencies: [SUB-001, SUB-002]
      estimated_complexity: medium
      acceptance_criteria: [AC-3, AC-4]
      
    - task_id: SUB-004
      name: 编写 API 文档
      type: docs
      specialist: docs
      dependencies: [SUB-001]
      estimated_complexity: low
      
  execution_order:
    parallel:
      - [SUB-001, SUB-002]  # 可并行执行
    sequential:
      - SUB-003  # 依赖 SUB-001, SUB-002
      - SUB-004  # 依赖 SUB-001
      
  timeline:
    estimated: 2 hours
    milestones:
      - 代码完成: 1 hour
      - 测试完成: 1.5 hours
      - 文档完成: 2 hours
```

---

## 3. Execution（执行记录）

### 定义

```yaml
execution:
  started_at: [开始时间]
  completed_at: [完成时间]
  
  subtasks:
    - task_id: SUB-001
      status: completed | failed | blocked
      specialist: [执行者]
      started_at: [开始时间]
      completed_at: [完成时间]
      
      actions:
        - action: [执行动作]
          timestamp: [时间戳]
          result: [结果]
          
      outputs:
        - type: code | test | docs
          path: [文件路径]
          description: [描述]
          
      issues:
        - type: warning | error | blocker
          description: [问题描述]
          resolution: [解决方案]
          
    - task_id: SUB-002
      ...
```

### 示例

```yaml
execution:
  started_at: 2024-01-15T10:05:00Z
  completed_at: 2024-01-15T11:45:00Z
  
  subtasks:
    - task_id: SUB-001
      status: completed
      specialist: Specialist-A (code)
      started_at: 2024-01-15T10:05:00Z
      completed_at: 2024-01-15T10:35:00Z
      
      actions:
        - action: 创建 src/api/login.py
          timestamp: 2024-01-15T10:10:00Z
          result: 成功
        - action: 实现登录逻辑
          timestamp: 2024-01-15T10:25:00Z
          result: 成功
        - action: 添加路由配置
          timestamp: 2024-01-15T10:35:00Z
          result: 成功
          
      outputs:
        - type: code
          path: src/api/login.py
          description: 登录 API 实现
        - type: code
          path: src/routes/__init__.py
          description: 路由配置更新
          
    - task_id: SUB-002
      status: completed
      specialist: Specialist-B (code)
      started_at: 2024-01-15T10:05:00Z
      completed_at: 2024-01-15T10:20:00Z
      
      outputs:
        - type: code
          path: src/utils/crypto.py
          description: 密码加密工具
          
    - task_id: SUB-003
      status: completed
      specialist: Specialist-C (test)
      started_at: 2024-01-15T10:40:00Z
      completed_at: 2024-01-15T11:15:00Z
      
      outputs:
        - type: test
          path: tests/test_login.py
          description: 登录功能单元测试
          
      issues:
        - type: warning
          description: 初始测试覆盖率 75%
          resolution: 添加边界测试用例，提升至 85%
```

---

## 4. Evidence（验证证据）

### 定义

```yaml
evidence:
  collected_by: Evaluator
  collected_at: [时间戳]
  
  verification_items:
    - ac_id: AC-1
      status: pass | fail
      verification_method: [验证方法]
      
      proof:
        type: file | command | test | data
        content: [证据内容]
        timestamp: [时间戳]
        
    - ac_id: AC-2
      ...
      
  test_results:
    command: [测试命令]
    output: [测试输出]
    summary:
      total: [总数]
      passed: [通过数]
      failed: [失败数]
      coverage: [覆盖率]
      
  lint_results:
    command: [Lint 命令]
    output: [Lint 输出]
    errors: [错误数]
    warnings: [警告数]
    
  files_verified:
    - path: [文件路径]
      exists: true | false
      checksum: [文件校验和]
```

### 示例

```yaml
evidence:
  collected_by: Evaluator
  collected_at: 2024-01-15T11:30:00Z
  
  verification_items:
    - ac_id: AC-1
      status: pass
      verification_method: 读取文件
      
      proof:
        type: file
        content: |
          # src/api/login.py
          @router.post("/login")
          async def login(credentials: LoginRequest):
              ...
        timestamp: 2024-01-15T11:30:05Z
        
    - ac_id: AC-2
      status: pass
      verification_method: 读取代码
      
      proof:
        type: file
        content: |
          # src/utils/crypto.py
          import bcrypt
          
          def hash_password(password: str) -> str:
              return bcrypt.hashpw(...)
        timestamp: 2024-01-15T11:30:10Z
        
    - ac_id: AC-3
      status: pass
      verification_method: 运行测试
      
      proof:
        type: test
        content: |
          def test_login_returns_valid_token():
              response = client.post("/login", json={...})
              assert response.status_code == 200
              token = response.json()["token"]
              assert verify_jwt(token) is True
        timestamp: 2024-01-15T11:30:15Z
        
    - ac_id: AC-4
      status: pass
      verification_method: pytest --cov
      
      proof:
        type: command
        content: |
          $ pytest --cov=src/api --cov=src/utils
          Coverage: 85%
        timestamp: 2024-01-15T11:30:20Z
        
    - ac_id: AC-5
      status: pass
      verification_method: ruff check
      
      proof:
        type: command
        content: |
          $ ruff check src/
          All checks passed!
        timestamp: 2024-01-15T11:30:25Z
        
  test_results:
    command: pytest tests/test_login.py -v
    output: |
      tests/test_login.py::test_login_success PASSED
      tests/test_login.py::test_login_invalid_password PASSED
      tests/test_login.py::test_login_user_not_found PASSED
      tests/test_login.py::test_token_validation PASSED
      4 passed in 0.52s
    summary:
      total: 4
      passed: 4
      failed: 0
      coverage: 85%
      
  lint_results:
    command: ruff check src/
    output: All checks passed!
    errors: 0
    warnings: 0
    
  files_verified:
    - path: src/api/login.py
      exists: true
      checksum: abc123...
    - path: src/utils/crypto.py
      exists: true
      checksum: def456...
    - path: tests/test_login.py
      exists: true
      checksum: ghi789...
```

---

## 5. Verdict（最终判决）

### 定义

```yaml
verdict:
  issued_by: Evaluator
  issued_at: [时间戳]
  
  result: PASS | FAIL | BLOCKED
  
  summary:
    total_acs: [总 AC 数]
    passed_acs: [通过数]
    failed_acs: [失败数]
    
  details:
    - ac_id: AC-1
      status: pass
      evidence_ref: [证据引用]
      
    - ac_id: AC-2
      status: pass
      evidence_ref: [证据引用]
      
    - ac_id: AC-3
      status: fail
      reason: [失败原因]
      
  blocking_issues:  # 仅 FAIL/BLOCKED 时
    - issue: [问题描述]
      impact: [影响]
      suggested_action: [建议行动]
      
  sign_off:
    evaluator: [Evaluator ID]
    confidence: high | medium | low
    notes: [备注]
```

### 示例 (PASS)

```yaml
verdict:
  issued_by: Evaluator-A
  issued_at: 2024-01-15T11:35:00Z
  
  result: PASS
  
  summary:
    total_acs: 5
    passed_acs: 5
    failed_acs: 0
    
  details:
    - ac_id: AC-1
      status: pass
      evidence_ref: evidence.verification_items[0]
      
    - ac_id: AC-2
      status: pass
      evidence_ref: evidence.verification_items[1]
      
    - ac_id: AC-3
      status: pass
      evidence_ref: evidence.verification_items[2]
      
    - ac_id: AC-4
      status: pass
      evidence_ref: evidence.verification_items[3]
      
    - ac_id: AC-5
      status: pass
      evidence_ref: evidence.verification_items[4]
      
  sign_off:
    evaluator: Evaluator-A
    confidence: high
    notes: |
      所有验收标准均已达成。
      代码质量良好，测试覆盖充分。
      可以交付给用户。
```

### 示例 (FAIL)

```yaml
verdict:
  issued_by: Evaluator-A
  issued_at: 2024-01-15T11:35:00Z
  
  result: FAIL
  
  summary:
    total_acs: 5
    passed_acs: 4
    failed_acs: 1
    
  details:
    - ac_id: AC-1
      status: pass
      evidence_ref: evidence.verification_items[0]
      
    - ac_id: AC-2
      status: pass
      evidence_ref: evidence.verification_items[1]
      
    - ac_id: AC-3
      status: pass
      evidence_ref: evidence.verification_items[2]
      
    - ac_id: AC-4
      status: fail
      reason: 测试覆盖率为 65%，低于要求的 80%
      
    - ac_id: AC-5
      status: pass
      evidence_ref: evidence.verification_items[4]
      
  blocking_issues:
    - issue: 测试覆盖率不足
      impact: 可能存在未测试的边界情况
      suggested_action: 添加更多测试用例，覆盖边界情况
      
  sign_off:
    evaluator: Evaluator-A
    confidence: high
    notes: |
      AC-4 未达成，测试覆盖率需要提升。
      建议 Planner 安排 Specialist 补充测试用例。
```

---

## Campaign 完整示例

```yaml
campaign:
  id: CAMP-001
  name: 用户登录功能
  status: completed
  
  task:
    id: TASK-001
    type: feature
    priority: high
    description:
      background: 用户需要登录系统才能使用个性化功能
      objective: 实现用户登录功能
      scope: [登录 API, 密码加密, Token 生成]
    acceptance_criteria:
      - { id: AC-1, description: POST /api/login 端点存在 }
      - { id: AC-2, description: 密码使用 bcrypt 加密 }
      - { id: AC-3, description: 返回有效的 JWT Token }
      - { id: AC-4, description: 测试覆盖率 >= 80% }
      - { id: AC-5, description: Lint 通过 }
      
  plan:
    created_by: Planner
    decomposition:
      - { task_id: SUB-001, name: 实现登录 API, specialist: code }
      - { task_id: SUB-002, name: 实现密码加密, specialist: code }
      - { task_id: SUB-003, name: 编写单元测试, specialist: test }
    execution_order:
      parallel: [[SUB-001, SUB-002]]
      sequential: [SUB-003]
      
  execution:
    started_at: 2024-01-15T10:05:00Z
    completed_at: 2024-01-15T11:45:00Z
    subtasks:
      - { task_id: SUB-001, status: completed, outputs: [src/api/login.py] }
      - { task_id: SUB-002, status: completed, outputs: [src/utils/crypto.py] }
      - { task_id: SUB-003, status: completed, outputs: [tests/test_login.py] }
      
  evidence:
    collected_by: Evaluator
    verification_items:
      - { ac_id: AC-1, status: pass, proof: 文件存在 }
      - { ac_id: AC-2, status: pass, proof: 使用 bcrypt }
      - { ac_id: AC-3, status: pass, proof: 测试通过 }
      - { ac_id: AC-4, status: pass, proof: 覆盖率 85% }
      - { ac_id: AC-5, status: pass, proof: Lint 通过 }
      
  verdict:
    issued_by: Evaluator
    result: PASS
    summary: { total_acs: 5, passed_acs: 5, failed_acs: 0 }
```

---

## Campaign 状态流转

```
┌─────────┐     开始执行      ┌─────────────┐
│ pending │ ───────────────► │ in_progress │
└─────────┘                  └─────────────┘
                                   │
                    ┌──────────────┼──────────────┐
                    │              │              │
                    ▼              ▼              ▼
              ┌───────────┐ ┌───────────┐ ┌───────────┐
              │ completed │ │   failed  │ │  blocked  │
              │  (PASS)   │ │  (FAIL)   │ │           │
              └───────────┘ └───────────┘ └───────────┘
                    │              │              │
                    ▼              │              │
                 交付             └──────► 修复后重新执行
```

---

## 文件存储

Campaign 目录结构：

```
campaigns/
├── 2026-05-17-task-name/           # 日期-任务名称
│   ├── task.md                     # 任务定义
│   ├── plan.md                     # 执行计划
│   ├── execution.md                # 执行记录
│   ├── evidence.md                 # 验证证据
│   ├── verdict.md                  # 最终判决
│   │
│   └── artifacts/                  # 执行产物目录
│       ├── phase-1/                # Phase 1 产物
│       │   ├── execution.log       # 执行日志
│       │   ├── test-output.log     # 测试输出
│       │   └── screenshots/        # 截图（如有）
│       │
│       ├── phase-2/                # Phase 2 产物
│       │   ├── execution.log
│       │   └── ...
│       │
│       └── final/                  # 最终产物
│           ├── test-results.xml    # 测试结果
│           └── coverage-report/    # 覆盖率报告
│
├── 2026-05-18-another-task/
│   └── ...
```

### artifacts 目录说明

| 目录 | 内容 | 创建者 |
|------|------|--------|
| `artifacts/phase-N/` | 第 N 阶段的执行产物 | Specialist |
| `artifacts/phase-N/execution.log` | 命令执行日志 | Specialist |
| `artifacts/phase-N/test-output.log` | 测试输出 | Specialist |
| `artifacts/final/` | 最终产物 | Evaluator |

### 日志文件格式

**execution.log 格式**:
```
[2026-05-17 10:00:00] Phase 1 开始
[2026-05-17 10:00:05] 执行: uv lock
[2026-05-17 10:00:10] 输出: Resolved 42 packages
[2026-05-17 10:00:15] 执行: uv sync
[2026-05-17 10:00:30] 输出: Installed 42 packages
[2026-05-17 10:00:35] Phase 1 完成
```

**test-output.log 格式**:
```
$ uv run pytest tests/ -v
tests/test_login.py::test_login_success PASSED
tests/test_login.py::test_login_invalid PASSED
2 passed in 0.52s
```

---

## 阶段性工作流（重要）

> ⚠️ **关键原则**: 文档必须按阶段创建，不能一次性创建所有文档！

### 工作流程图

```
┌─────────────────────────────────────────────────────────────┐
│                        Planner                               │
│  Step 1: 创建 task.md (任务定义)                              │
│  Step 2: 创建 plan.md (执行计划)                              │
│  Step 3: 委托 Specialist 执行 Phase 1                         │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│                       Specialist                             │
│  Step 4: 执行 Phase 1 任务                                    │
│  Step 5: 创建/更新 execution.md (记录执行日志)                 │
│  Step 6: 汇报完成                                             │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│                       Evaluator                              │
│  Step 7: 验证 Phase 1 结果                                    │
│  Step 8: 创建/更新 evidence.md (记录验证证据)                  │
│  Step 9: 给出 PASS/FAIL 判决                                  │
└─────────────────────────────────────────────────────────────┘
                            ↓
                    循环执行 Phase 2-N
                            ↓
┌─────────────────────────────────────────────────────────────┐
│                       Evaluator                              │
│  Step 10: 创建 verdict.md (最终判决)                          │
└─────────────────────────────────────────────────────────────┘
```

### 文档创建时机

| 文档 | 创建者 | 创建时机 | 内容 |
|------|--------|----------|------|
| `task.md` | Planner | Campaign 开始时 | 任务定义 |
| `plan.md` | Planner | task.md 完成后 | 执行计划 |
| `execution.md` | Specialist | 每个 Phase 执行时 | 执行日志（逐步追加） |
| `evidence.md` | Evaluator | 每个 Phase 验证时 | 验证证据（逐步追加） |
| `verdict.md` | Evaluator | 所有 Phase 完成后 | 最终判决 |

### 错误示例

❌ **错误做法**: 一次性创建所有文档
```
# 这会导致：
# 1. 无法追踪执行过程
# 2. 无法展示阶段性日志
# 3. 违背工作流设计原则
```

✅ **正确做法**: 按阶段逐步创建
```
Phase 1 开始 → Specialist 创建 execution.md
Phase 1 完成 → Evaluator 创建 evidence.md
Phase 2 开始 → Specialist 追加 execution.md
Phase 2 完成 → Evaluator 追加 evidence.md
...
所有 Phase 完成 → Evaluator 创建 verdict.md
```

### 日志展示要求

每个阶段的日志必须包含：

1. **执行日志 (execution.md)**
   - 执行的命令
   - 命令输出
   - 遇到的问题
   - 解决方案

2. **验证日志 (evidence.md)**
   - 验证命令
   - 验证结果
   - 通过/失败状态
   - 证据截图或输出

---

## Memories（经验记录）

> ⚠️ **重要**: Planner 在开始新任务时必须读取 memories，避免重复犯错！

### 目录结构

```
agentic-souls/
└── memories/
    ├── 001-campaign-workflow-issue.md    # Campaign 工作流问题
    ├── 002-subagent-artifacts-path-issue.md  # sub_agent 路径问题
    └── ...                                # 更多经验记录
```

### Memory 文件格式

```markdown
# Memory: [问题标题]

## 问题 ID
`MEM-XXX`

## 发现时间
YYYY-MM-DD

## 问题描述
[问题现象和影响]

## 根因分析
[问题根因]

## 解决方案
[解决方案]

## 教训总结
[教训总结]

## 后续行动
- [ ] 需要更新的文档
- [ ] 需要添加的规则
```

### 使用方式

**Planner 在开始新任务时**：

1. 读取 `agentic-souls/memories/` 目录下的所有 memory 文件
2. 识别与当前任务相关的经验教训
3. 在委托任务时，将相关 memory 内容传递给 Specialist

**示例**：

```markdown
## 相关经验 (来自 memories)

### MEM-002: sub_agent artifacts 路径问题
- 问题: sub_agent 将文件放到错误位置
- 解决: 使用绝对路径
- 本次任务注意: 所有 artifacts 路径必须使用绝对路径

### MEM-001: Campaign 工作流问题
- 问题: 一次性创建所有文档
- 解决: 按阶段创建文档
- 本次任务注意: 只创建 task.md 和 plan.md
```

### Memory 触发条件

在以下情况下创建新的 memory：

1. **发现新问题** - 遇到之前未记录的问题
2. **找到新解决方案** - 发现更好的解决方法
3. **工作流改进** - 优化工作流程
4. **工具使用经验** - 总结工具使用技巧
