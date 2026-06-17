<div align="right">
  <a href="README.zh-CN.md">🇨🇳 中文</a>
</div>

# Agentic Souls

> Document-driven AI workflow system based on a three-role architecture

## Overview

Agentic Souls is a document-driven AI workflow system that achieves high-quality software development by defining three core roles: **Planner**, **Evaluator**, and **Specialist**.

### Core Philosophy

```
Planner: You are the delivery owner — you plan, decompose, and verify. You do NOT write implementation code.
Evaluator: You are an independent reviewer — you do not help, you do not suggest. You only judge whether AC is met.
Specialist: You are the implementer — you only execute the sub-task assigned to you, report when done, and never overstep.
```

## Directory Structure

```
agentic-souls/
├── souls/                    # Role definitions
│   ├── planner.md           # Planner
│   ├── evaluator.md         # Evaluator
│   └── specialist.md        # Specialist
│
├── workflows/               # Workflow definitions
│   ├── feature-development.md  # Feature development
│   ├── bug-fix.md             # Bug fix
│   ├── code-review.md         # Code review
│   ├── refactoring.md         # Code refactoring
│   ├── data-migration.md      # Data migration
│   ├── dependency-upgrade.md  # Dependency upgrade
│   ├── deployment.md          # Deployment & release
│   ├── hotfix.md              # Hotfix / emergency fix
│   └── self-analysis.md       # Self-analysis workflow
│
├── skills/                  # Skill definitions
│   ├── python-code.md       # Python coding
│   ├── test-writing.md      # Test writing
│   ├── api-development.md   # API development
│   ├── frontend-development.md  # Frontend development
│   └── database-ops.md      # Database operations
│
├── memories/                # Issue records and solutions
│   ├── 001-campaign-workflow-issue.md
│   ├── 002-subagent-artifacts-path-issue.md
│   ├── 003-investment-data-accuracy.md
│   ├── 004-self-analysis-workflow.md
│   └── 005-investment-advice-framework.md
│
├── campaigns/               # Campaign storage
│   └── .gitkeep
│
└── docs/                    # Documentation
    └── campaign.md         # Campaign definition
```

## Three-Role Architecture

### Planner

- **Responsibility**: Plan tasks, decompose work, delegate execution, verify delivery
- **Constraint**: Never write more than 20 lines of implementation code
- **Core Rules**:
  1. All implementation is delegated to Specialist via Task tool
  2. Must call Evaluator before completion; self-evaluation is forbidden
  3. Read output immediately after each Task before deciding next step

### Evaluator

- **Responsibility**: Independently verify whether acceptance criteria (AC) are met
- **Constraint**: No help, no suggestions, only verdicts
- **Core Rules**:
  1. Don't trust Planner's statements; verify independently
  2. Don't issue PASS without evidence — read files, run commands
  3. Only output verdicts within scope; no suggestions

### Specialist

- **Responsibility**: Execute sub-tasks assigned by Planner
- **Constraint**: No overstepping, no planning, no overall judgment
- **Core Rules**:
  1. Only execute assigned sub-tasks; don't expand scope
  2. Don't plan overall or judge completion
  3. Report immediately on completion: what was done, where the artifacts are

## Workflow

```
User Request
     │
     ▼
┌─────────────┐
│   Planner   │ ── Analyze requirements, decompose tasks
└─────────────┘
     │
     ▼
┌─────────────┐
│ Specialist  │ ── Execute sub-tasks, report results
└─────────────┘
     │
     ▼
┌─────────────┐
│  Evaluator  │ ── Independent verification, verdict
└─────────────┘
     │
     ▼
  PASS/FAIL
```

## Usage

### 1. Select Workflow

Choose the appropriate workflow for your task type:

- **Feature Development**: `workflows/feature-development.md`
- **Bug Fix**: `workflows/bug-fix.md`
- **Code Review**: `workflows/code-review.md`

### 2. Load Role

Ask AI to load the corresponding soul document:

```text
Please read souls/planner.md. You are now in the Planner role.
```

### 3. Execute Workflow

Follow the steps defined in the workflow:

```text
Please follow workflows/feature-development.md to implement the new feature.
```

### 4. Switch Roles

Switch between roles at different stages:

```text
[Planner completes task decomposition]
→ Delegate to Specialist for execution
→ Specialist reports results
→ Call Evaluator for verification
→ Planner decides next step based on verdict
```

## Example

### Implementing a New Feature

```text
User: Help me implement user login

[AI loads Planner soul]
Planner:
  Analysis:
    - Feature: User login
    - Input: username, password
    - Output: JWT Token
  
  Task Decomposition:
    - Task-1: Implement login API (delegate to Specialist-code)
    - Task-2: Implement password hashing (delegate to Specialist-code)
    - Task-3: Write tests (delegate to Specialist-test)
  
  → Delegate Task-1 to Specialist

[AI loads Specialist soul]
Specialist:
  Task: Implement login API
  Done: Created src/api/login.py
  Artifacts: src/api/login.py, tests/test_login.py
  Note: Need to configure JWT_SECRET

[AI switches back to Planner]
Planner:
  → Read Specialist output
  → Continue delegating Task-2, Task-3
  → All tasks complete, call Evaluator

[AI loads Evaluator soul]
Evaluator:
  Verification:
    - AC-1: ✅ API exists
    - AC-2: ✅ Password hashed
    - AC-3: ✅ Tests pass
  Verdict: PASS
  Evidence: [file list, test output]

[AI switches back to Planner]
Planner:
  → Receive PASS verdict
  → Deliver to user: User login feature is complete
```

## Extension

### Add New Workflow

1. Create a new `.md` file in `workflows/` directory
2. Define workflow steps, roles, and acceptance criteria
3. Follow existing workflow format

### Add New Skill

1. Create a new `.md` file in `skills/` directory
2. Define skill capabilities, execution guidelines, and output templates
3. Follow existing skill format

### Custom Roles

1. Create a new role document in `souls/` directory
2. Define role responsibilities, behavior rules, and output templates
3. Update workflows to use the new role

## Quality Assurance

### Evidence-Driven

All verdicts must be supported by evidence:

- File paths
- Command outputs
- Test results
- Data comparisons

### Independent Verification

Evaluator must verify independently, without trusting Planner's statements:

```yaml
Verification Methods:
  - Read files: Check code exists
  - Run commands: Check functionality works
  - Check logs: Verify execution results
  - Compare data: Validate calculation accuracy
```

### Role Boundaries

Each role has clear boundary constraints:

- Planner: Does not write implementation code
- Evaluator: Does not give suggestions
- Specialist: Does not overstep into planning

## Best Practices

1. **Clear Acceptance Criteria**: Every task has well-defined AC
2. **Timely Reporting**: Specialist reports immediately upon completion
3. **Independent Verification**: Must pass through Evaluator verification
4. **Evidence Preservation**: Save all verification results to evidence/
5. **Continuous Improvement**: Optimize workflows based on feedback

## Integration

Agentic Souls can be integrated with any project:

```yaml
Integration Options:
  1. Use as standalone project
  2. Reference souls/ directory within project
  3. Customize workflows and skills
```

## References

- [Campaign Definition](docs/campaign.md)

## License

MIT
