---
name: subagent-driven-development
description: "Execute plans via delegate_task subagents (2-stage review)."
version: 1.1.0
author: Hermes Agent (adapted from obra/superpowers)
license: MIT
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: [delegation, subagent, implementation, workflow, parallel]
    related_skills: [writing-plans, requesting-code-review, test-driven-development]
---

# Subagent-Driven Development

## Overview

Execute implementation plans by dispatching fresh subagents per task with systematic two-stage review.

**Core principle:** Fresh subagent per task + two-stage review (spec then quality) = high quality, fast iteration.

## When to Use

Use this skill when:
- You have an implementation plan (from writing-plans skill or user requirements)
- Tasks are mostly independent
- Quality and spec compliance are important
- You want automated review between tasks

**vs. manual execution:**
- Fresh context per task (no confusion from accumulated state)
- Automated review process catches issues early
- Consistent quality checks across all tasks
- Subagents can ask questions before starting work

## The Process

### 1. Read and Parse Plan

Read the plan file. Extract ALL tasks with their full text and context upfront. Create a todo list:

```python
# Read the plan
read_file("docs/plans/feature-plan.md")

# Create todo list with all tasks
todo([
    {"id": "task-1", "content": "Create User model with email field", "status": "pending"},
    {"id": "task-2", "content": "Add password hashing utility", "status": "pending"},
    {"id": "task-3", "content": "Create login endpoint", "status": "pending"},
])
```

**Key:** Read the plan ONCE. Extract everything. Don't make subagents read the plan file — provide the full task text directly in context.

### 2. Per-Task Workflow

For EACH task in the plan:

#### Step 1: Dispatch Implementer Subagent

Use `delegate_task` with complete context:

```python
delegate_task(
    goal="Implement Task 1: Create User model with email and password_hash fields",
    context="""
    TASK FROM PLAN:
    - Create: src/models/user.py
    - Add User class with email (str) and password_hash (str) fields
    - Use bcrypt for password hashing
    - Include __repr__ for debugging

    FOLLOW TDD:
    1. Write failing test in tests/models/test_user.py
    2. Run: pytest tests/models/test_user.py -v (verify FAIL)
    3. Write minimal implementation
    4. Run: pytest tests/models/test_user.py -v (verify PASS)
    5. Run: pytest tests/ -q (verify no regressions)
    6. Commit: git add -A && git commit -m "feat: add User model with password hashing"

    PROJECT CONTEXT:
    - Python 3.11, Flask app in src/app.py
    - Existing models in src/models/
    - Tests use pytest, run from project root
    - bcrypt already in requirements.txt
    """,
    toolsets=['terminal', 'file']
)
```

#### Step 2: Dispatch Spec Compliance Reviewer

After the implementer completes, verify against the original spec:

```python
delegate_task(
    goal="Review if implementation matches the spec from the plan",
    context="""
    ORIGINAL TASK SPEC:
    - Create src/models/user.py with User class
    - Fields: email (str), password_hash (str)
    - Use bcrypt for password hashing
    - Include __repr__

    CHECK:
    - [ ] All requirements from spec implemented?
    - [ ] File paths match spec?
    - [ ] Function signatures match spec?
    - [ ] Behavior matches expected?
    - [ ] Nothing extra added (no scope creep)?

    OUTPUT: PASS or list of specific spec gaps to fix.
    """,
    toolsets=['file']
)
```

**If spec issues found:** Fix gaps, then re-run spec review. Continue only when spec-compliant.

#### Step 3: Dispatch Code Quality Reviewer

After spec compliance passes:

```python
delegate_task(
    goal="Review code quality for Task 1 implementation",
    context="""
    FILES TO REVIEW:
    - src/models/user.py
    - tests/models/test_user.py

    CHECK:
    - [ ] Follows project conventions and style?
    - [ ] Proper error handling?
    - [ ] Clear variable/function names?
    - [ ] Adequate test coverage?
    - [ ] No obvious bugs or missed edge cases?
    - [ ] No security issues?

    OUTPUT FORMAT:
    - Critical Issues: [must fix before proceeding]
    - Important Issues: [should fix]
    - Minor Issues: [optional]
    - Verdict: APPROVED or REQUEST_CHANGES
    """,
    toolsets=['file']
)
```

**If quality issues found:** Fix issues, re-review. Continue only when approved.

#### Step 4: Mark Complete

```python
todo([{"id": "task-1", "content": "Create User model with email field", "status": "completed"}], merge=True)
```

### 3. Final Review

After ALL tasks are complete, dispatch a final integration reviewer:

```python
delegate_task(
    goal="Review the entire implementation for consistency and integration issues",
    context="""
    All tasks from the plan are complete. Review the full implementation:
    - Do all components work together?
    - Any inconsistencies between tasks?
    - All tests passing?
    - Ready for merge?
    """,
    toolsets=['terminal', 'file']
)
```

### 4. Verify and Commit

```bash
# Run full test suite
pytest tests/ -q

# Review all changes
git diff --stat

# Final commit if needed
git add -A && git commit -m "feat: complete [feature name] implementation"
```

## Task Granularity

**Each task = 2-5 minutes of focused work.**

**Too big:**
- "Implement user authentication system"

**Right size:**
- "Create User model with email and password fields"
- "Add password hashing function"
- "Create login endpoint"
- "Add JWT token generation"
- "Create registration endpoint"

## Red Flags — Never Do These

- Start implementation without a plan
- Skip reviews (spec compliance OR code quality)
- Proceed with unfixed critical/important issues
- Dispatch multiple implementation subagents for tasks that touch the same files
- Make subagent read the plan file (provide full text in context instead)
- Skip scene-setting context (subagent needs to understand where the task fits)
- Ignore subagent questions (answer before letting them proceed)
- Accept "close enough" on spec compliance
- Skip review loops (reviewer found issues → implementer fixes → review again)
- Let implementer self-review replace actual review (both are needed)
- **Start code quality review before spec compliance is PASS** (wrong order)
- Move to next task while either review has open issues

## Handling Issues

### If Subagent Asks Questions

- Answer clearly and completely
- Provide additional context if needed
- Don't rush them into implementation

### If Reviewer Finds Issues

- Implementer subagent (or a new one) fixes them
- Reviewer reviews again
- Repeat until approved
- Don't skip the re-review

### If Subagent Fails a Task

- Dispatch a new fix subagent with specific instructions about what went wrong
- Don't try to fix manually in the controller session (context pollution)

## Efficiency Notes

**Why fresh subagent per task:**
- Prevents context pollution from accumulated state
- Each subagent gets clean, focused context
- No confusion from prior tasks' code or reasoning

**Why two-stage review:**
- Spec review catches under/over-building early
- Quality review ensures the implementation is well-built
- Catches issues before they compound across tasks

**Cost trade-off:**
- More subagent invocations (implementer + 2 reviewers per task)
- But catches issues early (cheaper than debugging compounded problems later)

## Integration with Other Skills

### With writing-plans

This skill EXECUTES plans created by the writing-plans skill:
1. User requirements → writing-plans → implementation plan
2. Implementation plan → subagent-driven-development → working code

### With test-driven-development

Implementer subagents should follow TDD:
1. Write failing test first
2. Implement minimal code
3. Verify test passes
4. Commit

Include TDD instructions in every implementer context.

### With requesting-code-review

The two-stage review process IS the code review. For final integration review, use the requesting-code-review skill's review dimensions.

### With systematic-debugging

If a subagent encounters bugs during implementation:
1. Follow systematic-debugging process
2. Find root cause before fixing
3. Write regression test
4. Resume implementation

## Example Workflow

```
[Read plan: docs/plans/auth-feature.md]
[Create todo list with 5 tasks]

--- Task 1: Create User model ---
[Dispatch implementer subagent]
  Implementer: "Should email be unique?"
  You: "Yes, email must be unique"
  Implementer: Implemented, 3/3 tests passing, committed.

[Dispatch spec reviewer]
  Spec reviewer: ✅ PASS — all requirements met

[Dispatch quality reviewer]
  Quality reviewer: ✅ APPROVED — clean code, good tests

[Mark Task 1 complete]

--- Task 2: Password hashing ---
[Dispatch implementer subagent]
  Implementer: No questions, implemented, 5/5 tests passing.

[Dispatch spec reviewer]
  Spec reviewer: ❌ Missing: password strength validation (spec says "min 8 chars")

[Implementer fixes]
  Implementer: Added validation, 7/7 tests passing.

[Dispatch spec reviewer again]
  Spec reviewer: ✅ PASS

[Dispatch quality reviewer]
  Quality reviewer: Important: Magic number 8, extract to constant
  Implementer: Extracted MIN_PASSWORD_LENGTH constant
  Quality reviewer: ✅ APPROVED

[Mark Task 2 complete]

... (continue for all tasks)

[After all tasks: dispatch final integration reviewer]
[Run full test suite: all passing]
[Done!]
```

## Remember

```
Fresh subagent per task
Two-stage review every time
Spec compliance FIRST
Code quality SECOND
Never skip reviews
Catch issues early
```

**Quality is not an accident. It's the result of systematic process.**

## Further reading (load when relevant)

When the orchestration involves significant context usage, long review loops, or complex validation checkpoints, load these references for the specific discipline:

- **`references/context-budget-discipline.md`** — Four-tier context degradation model (PEAK / GOOD / DEGRADING / POOR), read-depth rules that scale with context window size, and early warning signs of silent degradation. Load when a run will clearly consume significant context (multi-phase plans, many subagents, large artifacts).
- **`references/gates-taxonomy.md`** — The four canonical gate types (Pre-flight, Revision, Escalation, Abort) with behavior, recovery, and examples. Load when designing or reviewing any workflow that has validation checkpoints — use the vocabulary explicitly so each gate has defined entry, failure behavior, and resumption rules.

Both references adapted from gsd-build/get-shit-done (MIT © 2025 Lex Christopherson).

## 上下文注入 — AGENTS.md 目录规则

当 `delegate_task` 将子 Agent 派往特定目录时，先检查该目录及其上级目录是否存在 `AGENTS.md` 或 `.hermes-rules` 规则文件，将规则自动注入子 Agent 的 context，确保子 Agent 按该目录的约定行事。

### 检查流程

每次 `delegate_task` 调用前执行：

```bash
# 检查目标目录下的规则文件
ls <target_dir>/AGENTS.md 2>/dev/null || ls <target_dir>/.hermes-rules 2>/dev/null
```

如果文件存在，将内容拼接到 `delegate_task` 的 `context` 参数中。

### 完整调用示例

```python
target_dir = "~/projects/my-app/src"

# 检查规则文件
rules = ""
if path_exists(f"{target_dir}/AGENTS.md"):
    rules = read_file(f"{target_dir}/AGENTS.md")
elif path_exists(f"{target_dir}/.hermes-rules"):
    rules = read_file(f"{target_dir}/.hermes-rules")

# 注入到 delegate_task
delegate_task(
    goal="在 src/ 下添加用户注册 API 路由",
    context=f"工作目录: {target_dir}\n\n目录规则:\n{rules}",
    toolsets=["terminal", "file"]
)
```

### 多级目录规则合并

当子 Agent 进入深层子目录时，按以下优先级合并（离目标越近优先级越高）：

1. 目标目录下的 AGENTS.md（最高优先级）
2. 目标目录的父目录下的 AGENTS.md
3. 项目根目录下的 AGENTS.md（最低优先级）

重复字段以高优先级为准，不重复的字段全部保留。

### 规则文件内容示例

```markdown
# 目录规则

## 编码规范
- 使用 TypeScript，严格模式
- 缩进 2 空格，行尾无分号
- 所有公共函数必须有 JSDoc 注释

## 测试
- 每个函数至少一个测试用例
- 测试文件放在 __tests__/ 目录

## 约束
- 不允许使用 any 类型
- API 请求必须走 src/api/ 封装的客户端
```

### 额外配置

可在项目根目录放 `.hermes-agent.yaml` 定义全局约定：

```yaml
agents:
  default_workdir: "~/projects/my-app"
  always_inject_rules: true
  rules_files:
    - AGENTS.md
    - .hermes-rules
```

### 关键原则

- **永远在派发子 Agent 前检查规则** — 子 Agent 不了解目录规矩，必须注入。
- **多级合并时不覆盖** — 离目标越近的规则优先级越高，但远层的规则只要不冲突就保留。
- **规则注入属于子 Agent 的强制 context** — 不依赖子 Agent 自行发现。

## 代码清理模式 — 并行 3-Agent Simplify

在实现完成后，使用三个聚焦的 reviewer 子 Agent 并行审查最近的代码变更，聚合并应用合理的修复。三个窄 reviewer 优于一个宽 reviewer — 每个 reviewer 深入搜索代码库寻找单一类别的问题，三者并发运行。

### 触发条件

用户明确说出以下之一时才使用（不自动运行，因为消耗 3 个子 Agent 的 token）：
- "simplify" / "simplify my changes" / "clean up my changes"
- "review my code" / "review my recent changes"

可选修饰符：
- **Focus:** `"simplify focus on efficiency"` → 只跑 efficiency reviewer
- **Dry run:** `"just report"` → 只报告，不修改
- **Scope:** `"simplify the last commit"` / `"simplify src/foo.py"` → 限定 diff 范围

### Phase 1 — 识别变更

```bash
# 默认：未提交的工作区变更
git diff

# 如果为空，包含暂存变更
git diff HEAD

# 用户指定范围的变体：
git diff --staged              # "仅暂存区"
git diff HEAD~1                # "最后一次提交"
git diff main...HEAD           # "当前分支"
git diff -- src/foo.py         # 特定文件
```

如果 diff 非常庞大（>2000 行变更），警告用户 token 消耗会很高，建议缩小范围。

### Phase 2 — 并行启动三个 Reviewer

使用 `delegate_task` **批量模式**，一次性传递三个任务并发执行。每个 reviewer 接收**完整 diff**（不是片段），附带仓库绝对路径，分配 `terminal`、`file`、`search` 工具集。

**每个 reviewer 必须：**
- 搜索现有代码库寻找证据（不能仅从 diff 推理）
- 应用 Chesterton's Fence：标记前先运行 `git blame` 理解代码存在的理由；无法确定原始意图时标记 `confidence: low`
- 按结构化格式报告：
  ```
  file:line → problem → suggested fix | confidence: high/medium/low | risk: SAFE/CAREFUL/RISKY
  ```
- 跳过 nits 和纯风格改动 — 只标记有实质改善潜力的内容

**风险分级：**
- **SAFE** = 已证明不影响行为（unused imports, commented-out code, pass-through wrappers），可直接应用
- **CAREFUL** = 改善但不改语义（重命名局部变量、展平嵌套三元、提取辅助函数），需要测试验证后应用
- **RISKY** = 可能改变行为或破坏公共契约（N+1 重构、公共 API 改名、内存生命周期变更），只做标记供人工审核，不自动应用

**三个 Reviewer 的目标：**

**Reviewer 1 — 代码复用（Code Reuse）**
> 审查 diff 中是否存在与代码库已有功能重复的代码。搜索工具模块、共享辅助函数和相邻文件，找出新代码可以调用而非重写的已有函数、常量或模式。标记：重复已有功能的新函数；手工实现已有工具已覆盖的逻辑（手动字符串/路径处理、自定义环境检查、临时类型守卫、重新实现的解析器）。每一条都要指明应使用的已有功能及其位置。

**Reviewer 2 — 代码质量（Code Quality）**
> 审查 diff 中的质量问题。关注：冗余状态（重复或可从已有状态推导的值；不必要的缓存）；参数膨胀（不断追加的参数而不是重构函数结构）；带差异的复制粘贴（应共享抽象的近似重复块）；泄露的抽象（暴露内部实现、破坏已有封装边界）；字符串化代码（应使用常量/enum/注册表的原始字符串）；AI 生成痕迹模式（多余注释重复显而易见的代码、对已验证输入的不必要防御性 null 检查、绕过类型系统的 `as any` 转换、与文件其余部分风格不一致的模式）。每一条给出具体重构方案。

**Reviewer 3 — 效率（Efficiency）**
> 审查 diff 中的效率问题。关注：不必要的工作（冗余计算、重复文件读取、重复 API 调用、N+1 访问模式）；错过的并发机会（独立操作顺序执行）；热路径膨胀（启动或请求路径上的重型/阻塞操作）；TOCTOU 反模式（操作前先做存在检查而不是直接操作并处理错误）；内存问题（无界增长、缺少清理、监听器/句柄泄露）；过于宽泛的读取（只需要切片却读取整个文件）；静默失败（空的 catch 块、忽略错误返回、`except: pass`、无处理的 `.catch(() => {})`、错误传播缺口 — 至少应在吞没前记录日志）。每一条给出具体修复方案及为什么更快或更安全。

### Phase 3 — 聚合与应用

等待三个 reviewer 全部返回。

1. **合并** 结果列表，去除 reviewer 之间的重复。
2. **丢弃误报** — 你有最大上下文权限；不必与 reviewer 争论，直接沉默丢弃弱或错误的建议。
3. **解决冲突**。Reviewer 之间可能意见不同。默认解决顺序：**正确性 > 用户声明的 focus > 可读性/复用 > 微性能优化**。不要应用损害清晰度以换取性能的"修复"，除非路径确实很热。
4. **按风险层级应用：**
   - **SAFE 优先**（自动应用）：unused imports, 注释掉的代码, pass-through wrappers, 冗余类型断言。应用后跑测试。
   - **CAREFUL 其次**（带验证逐文件应用）：重命名局部变量、展平三元、提取辅助函数、合并重复。每个文件后跑测试。回滚任何破坏性修改。
   - **RISKY 最后**（只做标记 — 不自动应用）：N+1 重构、公共 API 更改、并发修复、错误处理变更。逐条附上风险描述和测试覆盖状态。
5. **验证**：针对修改涉及的文件跑目标测试（非全量套件），跑项目使用的 linter/类型检查。
6. **总结**：按 reviewer 类别和风险层级分组的已应用修复清单，以及任何有意跳过的发现及其原因。

### 关键原则

- **恰好 3 个 reviewer** — 更多意味着更高成本和更多冲突建议，不是更好覆盖。
- **给每个 reviewer 完整 diff** — 分割 diff 会破坏设计：跨文件重复和 N+1 问题只有全貌才能发现。
- **Reviewer 搜索，不猜测** — 复用建议若无法指向已有工具的 `file:line` 位置则为噪音。丢弃缺乏证据的发现。
- **应用 ≠ 重写** — 这是清理用户最近的更改，不是重构整个模块。编辑范围限定在 diff 涉及的内容加上修复所需的极小周边改写。
- **尊重项目约定** — 如果仓库有 AGENTS.md / CLAUDE.md / HERMES.md 或 linter 配置，将这些规则融入 reviewer prompt 以确保建议符合团队风格。
- **大 diff 会撑爆 context** — 如果 diff 过大，先缩小范围再委托。
- **改名前检查公共契约** — 导出名、API 路由路径、DB 列名和配置键是契约 — 即使名字不好，改名也会破坏消费者。公共契约变更一律标记为 RISKY。
- **不删除"不必要"的错误处理** — 空的 catch 块或被忽略的错误可能是故意的（该错误在该上下文中是预期且良性的）。标记，不删除；由人工决定。
