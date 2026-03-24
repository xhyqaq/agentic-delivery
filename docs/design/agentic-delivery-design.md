# Agentic Delivery — Skills 设计文档

> 端到端交付编排系统，通过 multi-agent 架构实现规范化的需求分析、代码实现、质量保障和文档沉淀，并支持跨会话的知识积累。

---

## 1. 设计目标

| 目标 | 说明 |
|------|------|
| 端到端交付 | 从需求识别到代码交付、review、文档沉淀，一条完整流水线 |
| 跨会话沉淀 | 每次交付产出的文档（设计、plan、changelog）持久化到项目中，后续会话可直接复用，减少重复上下文 |
| 质量保障 | 强制 review 机制，独立的 reviewer subagent 避免自查盲区 |
| 上下文最小化 | 每个 subagent 只接收完成其任务所需的最小信息，避免上下文污染 |
| 按需编排 | 大需求走完整流水线，小需求/Bug 走简化路径，不浪费时间 |

---

## 2. 架构总览

### 2.1 核心原则

- **主 agent 只负责编排和调度**，不直接执行实现任务
- **Subagent 执行具体任务**，每个 subagent 上下文隔离、职责单一
- **文档驱动**：每个阶段的产出都是文档（spec、plan、changelog），文档即沟通协议
- **Subagent 无状态**：每次 dispatch 都是全新实例，通过 prompt 注入上下文

### 2.2 Multi-Agent 角色清单

| 角色 | 职责 | 触发方式 |
|------|------|----------|
| **Orchestrator**（主 agent） | 意图识别、流程编排、subagent 调度、结果汇总 | 用户输入触发 |
| **Doc Scanner** | 扫描项目结构，生成/检查规范文档 | 主 agent dispatch |
| **Brainstormer** | 需求分析、方案探讨（主 agent 自身承担，不派 subagent） | 主 agent 执行 |
| **Spec Reviewer** | 审查设计文档的完整性和一致性 | 主 agent dispatch |
| **Plan Writer** | 将设计转化为详细实现计划（主 agent 自身承担） | 主 agent 执行 |
| **Plan Reviewer** | 审查实现计划 | 主 agent dispatch |
| **Implementer** | 按 task 实现代码，遵循 TDD | 主 agent dispatch |
| **Spec Compliance Reviewer** | 检查实现是否符合需求/设计 | 主 agent dispatch |
| **Code Quality Reviewer** | 检查代码质量、架构合理性 | 主 agent dispatch |
| **Fix Agent** | 修复 review 发现的问题 | 主 agent dispatch（按分级策略选择角色） |
| **Doc Syncer** | review 通过后同步更新项目文档 | 主 agent dispatch |

---

## 3. 工作流编排

### 3.1 完整流水线（大需求）

```
用户输入
  │
  ▼
┌─────────────────────────────────┐
│ Stage 1: 意图识别（主 agent）      │
│ 判断需求规模 → 选择路径            │
└──────────────┬──────────────────┘
               │ 大需求
               ▼
┌─────────────────────────────────┐
│ Stage 2: 文档扫描与沉淀           │
│ dispatch: Doc Scanner subagent   │
│ 产出: docs/<project>/project-context.md │
└──────────────┬──────────────────┘
               ▼
┌─────────────────────────────────┐
│ Stage 3: 需求分析与设计           │
│ 主 agent 执行 brainstorming      │
│ dispatch: Spec Reviewer subagent │
│ 产出: docs/<project>/<feature>/design-spec.md │
└──────────────┬──────────────────┘
               ▼
┌─────────────────────────────────┐
│ Stage 4: 实现计划编写             │
│ 主 agent 编写 plan               │
│ dispatch: Plan Reviewer subagent │
│ 产出: docs/<project>/<feature>/implementation-plan.md │
└──────────────┬──────────────────┘
               ▼
┌─────────────────────────────────┐
│ Stage 5: 代码实现                │
│ per-task dispatch Implementer    │
│ 独立任务可并行，依赖任务顺序执行    │
└──────────────┬──────────────────┘
               ▼
┌─────────────────────────────────┐
│ Stage 6: Review 循环             │
│ per-task:                        │
│   ├─ Spec Compliance Reviewer    │
│   ├─ Code Quality Reviewer       │
│   ├─ Fix（分级策略，见 §5）        │
│   └─ Doc Syncer                  │
│ 每个 task review 通过后 commit    │
└──────────────┬──────────────────┘
               ▼
┌─────────────────────────────────┐
│ Stage 7: 产出总结                │
│ 主 agent 汇总所有阶段结果         │
│ 输出给用户                       │
└─────────────────────────────────┘
```

### 3.2 各 Stage 详细说明

#### Stage 1: 意图识别

**执行者**：主 agent（不需要 subagent）

**职责**：
- 分析用户输入，判断需求规模
- 检查 `docs/` 目录下是否已有相关沉淀文档（跨会话复用）

**分类标准**：

| 类型 | 特征 | 路径 |
|------|------|------|
| 大需求 | 涉及多模块/多文件、需要设计、有架构影响 | 完整流水线 Stage 1-7 |
| 小需求 | 单文件/单函数级别、改动范围明确 | 简化路径（见 §3.3） |
| Bug 修复 | 明确的错误行为、需要定位根因 | 调试路径（见 §3.4） |

**跨会话复用逻辑**：
- 若 `docs/<project>/<feature>/` 已存在相关文档，读取后跳过已完成的 stage
- 例如：design-spec.md 已存在 → 跳过 Stage 3，直接进入 Stage 4

#### Stage 2: 文档扫描与沉淀

**执行者**：Doc Scanner subagent

**输入（最小上下文）**：
- 项目根目录路径
- 需要扫描的文件列表提示：`CLAUDE.md`、`AGENTS.md`、`docs/*.md`、`README.md`
- 文档模板规范（见 §6）

**工作内容**：
1. 扫描项目结构（目录树、技术栈检测）
2. 读取已有文档（CLAUDE.md、AGENTS.md 等）
3. 必要时搜索代码提取关键信息（入口文件、核心模块、依赖关系）
4. 按模板生成 `docs/<project>/project-context.md`

**产出**：`docs/<project>/project-context.md`

**生命周期**：用完即关。产出已持久化到文件。

#### Stage 3: 需求分析与设计

**执行者**：主 agent（brainstorming 流程）

**流程**（参考 superpowers/brainstorming）：
1. 读取 Stage 2 产出的 project-context.md
2. 每次向用户提一个问题，澄清需求
3. 提出 2-3 种方案，分析 trade-offs，给出推荐
4. 分段呈现设计，每段获取用户确认
5. 编写设计文档到 `docs/<project>/<feature>/design-spec.md`
6. Dispatch Spec Reviewer subagent 审查设计文档
7. Review loop：最多 3 轮，未通过则上报用户

**Spec Reviewer subagent 输入**：
- 设计文档路径
- 审查清单（完整性、一致性、清晰度、范围、YAGNI）

**Spec Reviewer 生命周期**：用完即关。

**设计原则**：
- 模块隔离：每个单元职责单一、接口清晰、可独立理解和测试
- YAGNI：不设计未被需求要求的功能
- 在已有代码库中优先遵循现有模式

#### Stage 4: 实现计划编写

**执行者**：主 agent

**输入**：
- `docs/<project>/<feature>/design-spec.md`
- `docs/<project>/project-context.md`

**工作内容**（参考 superpowers/writing-plans）：
1. 梳理文件结构：明确哪些文件新建、哪些文件修改、修改哪些行
2. 将设计拆解为 bite-sized tasks（每个 task 2-5 分钟）
3. 每个 task 包含：涉及的文件路径、具体步骤、测试命令、预期结果、commit 信息
4. 标注 task 之间的依赖关系（哪些可并行、哪些必须顺序）
5. Dispatch Plan Reviewer subagent 审查
6. Review loop：最多 3 轮

**并行编排硬性约束**：
> 如果两个 task 会修改同一个文件，则**禁止并行**，必须标记为顺序执行。
> 只有确认无文件交集的 task 才允许标记为可并行。
> 这是编排层的职责，在 plan 阶段就必须明确，不能留到执行阶段再处理。

**产出**：`docs/<project>/<feature>/implementation-plan.md`

**Plan 文档格式**：

```markdown
# [功能名] 实现计划

**目标**：[一句话描述]
**架构**：[2-3 句话描述方案]
**技术栈**：[关键技术/库]

---

## 文件结构

| 操作 | 文件路径 | 职责 |
|------|----------|------|
| 新建 | src/xxx.ts | ... |
| 修改 | src/yyy.ts:45-80 | ... |
| 测试 | tests/xxx.test.ts | ... |

## Task 依赖关系

- Task 1, 2: 无依赖，可并行
- Task 3: 依赖 Task 1
- Task 4, 5: 无依赖，可并行（前端/后端）

---

### Task 1: [组件名]

**文件**：
- 新建: `exact/path/to/file.ts`
- 测试: `tests/exact/path/to/test.ts`

**步骤**：
- [ ] Step 1: 编写失败测试
- [ ] Step 2: 运行测试确认失败
- [ ] Step 3: 编写最小实现
- [ ] Step 4: 运行测试确认通过
- [ ] Step 5: Commit
```

#### Stage 5: 代码实现

**执行者**：per-task Implementer subagent

**主 agent 职责**（编排）：
1. 读取 implementation-plan.md，一次性提取所有 task 文本
2. 按依赖关系决定执行顺序：
   - 无依赖的 task → 并行 dispatch（使用 `dispatching-parallel-agents` 模式）
   - 有依赖的 task → 顺序 dispatch
3. 为每个 subagent 构造最小上下文 prompt

**Implementer subagent 输入（最小上下文）**：
- Task 全文（从 plan 中复制，不让 subagent 读取 plan 文件）
- 该 task 相关的规范文档片段（不是全部文档）
- 工作目录路径
- 前置 task 的接口定义（如果有依赖）

**前后端并行示例**：
```
主 agent 提取 Task 4（后端）和 Task 5（前端）

后端 subagent 收到：
  - Task 4 全文
  - 后端规范文档
  - 接口定义（API schema）

前端 subagent 收到：
  - Task 5 全文
  - 前端规范文档
  - 接口定义（API schema）

两者并行执行，互不干扰
```

**Implementer 状态处理**：

| 状态 | 主 agent 行为 |
|------|--------------|
| DONE | 进入 Stage 6 review |
| DONE_WITH_CONCERNS | 评估 concerns，决定是否进入 review 还是先处理 |
| NEEDS_CONTEXT | 补充上下文后重新 dispatch |
| BLOCKED | 评估原因：上下文不足 → 补充；任务太复杂 → 拆分；plan 有误 → 回退 Stage 4 |

**Implementer 生命周期**：返回结果后关闭。如需 fix，dispatch 新的 subagent。

#### Stage 6: Review 循环

**执行者**：多个 reviewer subagent（per-task 执行）

**流程**：

```
Implementer 返回 DONE
       │
       ▼
  Spec Compliance Review（subagent）
       │
       ├── 通过 → Code Quality Review（subagent）
       │              │
       │              ├── 通过 → Commit → Doc Sync → 下一个 task
       │              │
       │              └── 不通过 → Fix（见分级策略 §5）→ 重新 Code Quality Review
       │
       └── 不通过 → Fix（见分级策略 §5）→ 重新 Spec Compliance Review
```

**顺序不可颠倒**：必须先 Spec Compliance，再 Code Quality。因为如果实现都不符合需求，讨论代码质量没有意义。

**Review 上限**：每个阶段最多 3 轮。超过 3 轮 → 上报用户决策。

**Commit 策略**：
- 每个 task 的 review 全部通过后立即 commit
- Fix 后也应 commit（fix commit 与原实现 commit 分开）

**Doc Syncer subagent**：
- 在该 task review 全部通过后 dispatch
- 输入：task 描述、变更文件列表、review 结果摘要
- 职责：更新 `docs/<project>/<feature>/changelog.md`
- 生命周期：用完即关

#### Stage 7: 产出总结

**执行者**：主 agent

**输出格式**：

```markdown
## 交付总结

### 执行概览
- 需求类型：大需求
- 总 Task 数：N
- 并行执行：Task X, Y
- Review 轮次：共 M 轮

### 各阶段产出

| 阶段 | 产出 | 状态 |
|------|------|------|
| 文档扫描 | docs/\<project\>/project-context.md | 已生成 |
| 需求设计 | docs/\<project\>/\<feature\>/design-spec.md | 已确认 |
| 实现计划 | docs/\<project\>/\<feature\>/implementation-plan.md | 已确认 |
| 代码实现 | [文件变更列表] | 已完成 |
| Review | 全部通过 | N 轮 fix |
| 文档同步 | docs/\<project\>/\<feature\>/changelog.md | 已更新 |

### 变更文件清单
- `src/xxx.ts` — 新建，[职责]
- `src/yyy.ts` — 修改，[改动点]
- `tests/xxx.test.ts` — 新建，[覆盖内容]
```

---

### 3.3 简化路径（小需求）

```
用户输入
  │
  ▼
Stage 1: 意图识别 → 判定为小需求
  │
  ▼
Stage 5: 代码实现（主 agent 可直接实现，不强制 subagent）
  │
  ▼
Stage 6: Review（简化为单阶段）
  │  dispatch 一个 Code Quality Reviewer subagent
  │  跳过 Spec Compliance Review（小需求无独立 spec）
  │
  ▼
Commit → Stage 7: 简要总结
```

**简化点**：
- 跳过 Stage 2（文档扫描）、Stage 3（brainstorming）、Stage 4（plan 编写）
- 实现阶段主 agent 可以自己做，不强制派 subagent
- Review 只做 Code Quality，不做 Spec Compliance

### 3.4 调试路径（Bug 修复）

```
用户输入
  │
  ▼
Stage 1: 意图识别 → 判定为 Bug 修复
  │
  ▼
Systematic Debugging（主 agent 执行）
  │  Phase 1: 根因调查
  │  Phase 2: 模式分析
  │  Phase 3: 假设验证
  │  Phase 4: 实现修复
  │
  ▼
Stage 6: Review（同小需求，单阶段 Code Quality Review）
  │
  ▼
Commit → Stage 7: 简要总结（含根因分析说明）
```

---

## 4. Subagent 生命周期管理

### 4.1 总览

| Subagent | 何时创建 | 何时销毁 | 是否需要复活 |
|----------|----------|----------|-------------|
| Doc Scanner | Stage 2 开始 | 文档生成完毕 | 否，产出已持久化 |
| Spec Reviewer | Stage 3 设计文档写完后 | 审查报告返回 | 可能，review loop 中每轮重新 dispatch |
| Plan Reviewer | Stage 4 plan 写完后 | 审查报告返回 | 同上 |
| Implementer | Stage 5 每个 task 开始 | 实现结果返回 | 否，fix 时 dispatch 新实例 |
| Spec Compliance Reviewer | Stage 6 实现完成后 | 审查报告返回 | 可能，re-review 时重新 dispatch |
| Code Quality Reviewer | Stage 6 Spec Review 通过后 | 审查报告返回 | 同上 |
| Fix Agent | Stage 6 review 不通过时 | 修复结果返回 | 否 |
| Doc Syncer | Stage 6 review 全部通过后 | 文档更新完毕 | 否 |

### 4.2 核心原则

1. **无状态 dispatch**：每次 dispatch 都是全新的 subagent 实例，通过 prompt 注入全部所需上下文
2. **产出持久化**：subagent 的产出必须写入文件或通过返回值传递给主 agent，不依赖 subagent 的内存
3. **不囤积 subagent**：任务完成就关闭，需要时重新创建。不存在"等待中"的 subagent

---

## 5. Review 分级修复策略

### 5.1 问题分级

当 Reviewer 返回问题时，主 agent 根据问题严重程度选择不同的修复策略：

```
Reviewer 返回问题
       │
       ▼
  主 agent 评估问题级别
       │
       ├── Minor / Important（局部修复）
       │     定义：逻辑小错、命名不当、缺少边界处理、
       │           魔法数字、缺少注释等
       │     策略：dispatch 新的 Implementer subagent
       │           输入 = review 报告 + 相关代码文件
       │           修完后 re-review
       │
       ├── Critical（设计方向性问题）
       │     定义：实现方案根本错误、违背设计 spec、
       │           架构层面缺陷、需要重写而非 patch
       │     策略：三选一
       │       ├─ 方案A：dispatch 独立 Fix subagent
       │       │   输入 = review 报告 + 原始 spec（不给原代码）
       │       │   从头实现，避免被原代码思路带偏
       │       ├─ 方案B：回退到 Stage 4 重新设计该 task
       │       └─ 方案C：上报用户决策
       │
       └── 超过 3 轮未通过
             策略：强制上报用户
             附带：所有轮次的 review 报告 + 修复尝试摘要
```

### 5.2 为什么不一律用独立 Fix subagent

- Minor/Important 级别问题：review 报告已经**精确指出了问题所在**，新 dispatch 的 subagent 拿到 review 报告 + 代码就够了。由于 Claude Code 的 Task tool 是无状态的，这个"新 implementer"本身就是一个全新的上下文，不存在思维惯性
- 只有 Critical 级别问题才需要"丢掉原代码从头来"，因为原代码的结构本身就是错的

### 5.3 安全网

- **re-review 不可省略**：任何修复后都必须由新的 Reviewer subagent 审查
- **re-reviewer 是新实例**：不是原来那个 reviewer，避免"审美疲劳"
- **最大轮次限制**：3 轮，防止无限循环
- **用户兜底**：超过 3 轮自动上报

---

## 6. 文档规范与模板

### 6.1 目录结构

```
docs/
  <project-name>/
    ├── project-context.md                 # Stage 2 产出：项目级上下文（唯一一份）
    │
    ├── <feature-or-module-A>/
    │   ├── design-spec.md                 # Stage 3 产出：需求设计文档
    │   ├── implementation-plan.md         # Stage 4 产出：实现计划
    │   └── changelog.md                   # Stage 6 产出：变更记录
    │
    └── <feature-or-module-B>/
        ├── design-spec.md
        ├── implementation-plan.md
        └── changelog.md
```

**说明**：
- `project-context.md` 是项目级别的，只有一份，放在 `docs/<project-name>/` 根下
- 每个需求/模块有独立的子目录，包含各自的 design-spec、plan、changelog
- 同一个项目的多个需求共享同一个 `project-context.md`

### 6.2 project-context.md 模板

```markdown
# 项目上下文

## 基本信息
- **项目名称**：
- **技术栈**：
- **包管理器**：
- **构建工具**：
- **测试框架**：

## 目录结构
[项目核心目录树，只到第二/三层]

## 核心模块
| 模块 | 路径 | 职责 |
|------|------|------|
| ... | ... | ... |

## 编码约定
[从 CLAUDE.md / AGENTS.md / 现有代码中提取的约定]

## 已知约束
[部署环境、兼容性要求、性能要求等]
```

### 6.3 design-spec.md 模板

```markdown
# [功能名] 设计文档

## 需求概述
[一段话描述要解决的问题和目标]

## 方案选型
### 方案 A: [名称]
- 优点：
- 缺点：

### 方案 B: [名称]
- 优点：
- 缺点：

### 选定方案：[X]
[选择理由]

## 详细设计

### 架构设计
[模块划分、模块间关系]

### 数据流
[数据如何在模块间流转]

### 接口定义
[模块间的接口 / API 定义]

### 错误处理
[异常场景及处理策略]

### 测试策略
[测试范围、关键测试用例]

## 影响范围
[现有代码中受影响的部分]
```

### 6.4 changelog.md 模板

```markdown
# 变更记录

## [日期] [功能名]

### Task 1: [名称]
- **变更文件**：`path/to/file.ts`
- **变更类型**：新建 / 修改
- **变更说明**：[具体改了什么]
- **Review 状态**：通过（N 轮）

### Task 2: [名称]
...
```

### 6.5 防漂移措施

1. **Doc Scanner 和 Doc Syncer 的 prompt 中嵌入完整模板**：不是给路径让 subagent 自己读，而是把模板文本直接写入 prompt
2. **固定 Section 结构**：模板中的每个 section 标题是固定的，subagent 只填充内容
3. **主 agent 验证**：Doc Syncer 返回结果后，主 agent 快速检查格式是否符合模板

---

## 7. 上下文最小化原则

### 7.1 核心规则

> 给 subagent 的应该是**完成其任务所需的最小信息集**，而不是主 agent 拥有的全部上下文。

### 7.2 各 Subagent 的上下文构造

| Subagent | 需要 | 不需要 |
|----------|------|--------|
| Doc Scanner | 项目根路径、扫描文件列表、模板 | 用户需求、设计文档 |
| Spec Reviewer | 设计文档路径、审查清单 | 项目代码、实现计划 |
| Plan Reviewer | plan 文档路径、设计文档路径 | 项目代码细节 |
| Implementer（后端） | task 全文、后端规范、接口定义 | 前端代码/规范、其他 task 内容 |
| Implementer（前端） | task 全文、前端规范、接口定义 | 后端代码/规范、其他 task 内容 |
| Spec Compliance Reviewer | task 需求文本、implementer 报告 | 其他 task、项目全局上下文 |
| Code Quality Reviewer | 变更的 git diff（BASE_SHA..HEAD_SHA）、task 描述 | 需求文档、其他 task |
| Fix Agent | review 报告、相关代码文件（Minor）或 原始 spec（Critical） | 其他 task、项目全局上下文 |
| Doc Syncer | task 描述、变更文件列表、changelog 模板 | 代码实现细节、review 过程 |

### 7.3 反模式

- **不要让 subagent 读取 plan 文件**：把 task 全文粘贴到 prompt 中
- **不要传递主 agent 的会话历史**：构造精确的 prompt
- **不要把所有文档都给同一个 subagent**：按角色筛选

---

## 8. 可复用的 Superpowers Skills

以下 skills 来自 [obra/superpowers](https://github.com/obra/superpowers)，可直接复用或基于其改造：

| Superpowers Skill | 对应本设计的 Stage | 复用方式 |
|-------------------|-------------------|----------|
| `brainstorming` | Stage 3 | 核心流程直接复用：单问题提问、2-3方案、分段确认、spec review loop |
| `writing-plans` | Stage 4 | 复用 bite-sized task 格式、文件结构设计、plan review loop |
| `subagent-driven-development` | Stage 5-6 | 复用 per-task dispatch 模式、两阶段 review、状态处理（DONE/BLOCKED/NEEDS_CONTEXT） |
| `dispatching-parallel-agents` | Stage 5 | 复用并行 dispatch 模式，用于前后端等独立 task |
| `requesting-code-review` | Stage 6 | 复用 code-reviewer prompt 模板 |
| `systematic-debugging` | 调试路径 | 直接复用四阶段调试法 |
| `verification-before-completion` | Stage 6-7 | 复用"证据先于断言"原则 |
| `finishing-a-development-branch` | Stage 7 后 | 如需分支管理，复用其完整流程 |

### 需要自行设计的 Skills

| Skill | 职责 | 原因 |
|-------|------|------|
| `intent-dispatcher` | 意图识别与路径选择 | superpowers 没有显式的分流机制 |
| `doc-scanner` | 项目文档扫描与生成 | superpowers 没有文档沉淀体系 |
| `doc-syncer` | review 后文档同步 | superpowers 没有这个环节 |
| `review-fix-strategy` | review 分级修复决策 | superpowers 统一由 implementer 修复，我们需要分级策略 |

---

## 9. Skill 入口与触发设计

### 9.1 触发方式：Skills Auto-Discovery

采用 skills 自动发现机制，而非手动 slash command。Claude 根据用户输入自动匹配 skill 的 `description` 字段来决定是否加载。

**主入口 Skill**：`agentic-delivery`

```yaml
---
name: agentic-delivery
description: "Use when user requests new features, enhancements, bug fixes, or any development task that involves code changes. Covers intent recognition, requirement analysis, design, implementation, code review, and documentation."
---
```

**触发后由主入口 skill 内部完成意图识别**（Stage 1），再路由到对应路径。

### 9.2 子 Skills 的加载

主入口 skill 在不同 stage 按需调用子 skills，子 skills 不需要被用户直接触发：

| 子 Skill | 被谁调用 | 加载时机 |
|----------|----------|----------|
| `doc-scanner` | 主 skill | Stage 2，需要扫描项目时 |
| `brainstorming` | 主 skill | Stage 3，大需求的需求分析时 |
| `writing-plans` | 主 skill | Stage 4，编写实现计划时 |
| `subagent-driven-development` | 主 skill | Stage 5-6，实现和 review 时 |
| `systematic-debugging` | 主 skill | Bug 修复路径时 |
| `doc-syncer` | 主 skill | Stage 6，review 通过后 |

---

## 10. 平台适配

### 10.1 设计原则

- **以 Claude Code 为主平台编写**：所有 skill 中使用 Claude Code 的工具名（Task、Read、Edit、Write、Bash 等）
- **通过映射层适配其他平台**：不在每个 skill 中写多套逻辑，而是维护一份工具映射表
- **Subagent 机制差异封装**：不同平台的 subagent 调用方式不同，在编排层统一封装

### 10.2 工具映射表（Codex）

> 直接复用自 [superpowers/codex-tools.md](https://github.com/obra/superpowers/blob/main/skills/using-superpowers/references/codex-tools.md)

Skills 统一使用 Claude Code 工具名编写。其他平台按此表映射：

| Skill 中引用的工具 | Codex 等效方式 |
|-----------------|------------------|
| `Task` tool (dispatch subagent) | `spawn_agent` |
| Multiple `Task` calls (parallel) | Multiple `spawn_agent` calls |
| Task returns result | `wait` |
| Task completes automatically | `close_agent` to free slot |
| `TodoWrite` (task tracking) | `update_plan` |
| `Skill` tool (invoke a skill) | Skills load natively — just follow the instructions |
| `Read`, `Write`, `Edit` (files) | Use your native file tools |
| `Bash` (run commands) | Use your native shell tools |

**Codex 需要开启 multi-agent 支持**：

```toml
# ~/.codex/config.toml
[features]
multi_agent = true
```

这会启用 `spawn_agent`、`wait`、`close_agent`，用于 `dispatching-parallel-agents` 和 `subagent-driven-development` 等 skills。

### 10.3 适配文件

在 skills 目录下维护一个 `references/` 目录：

```
skills/
  agentic-delivery/
    SKILL.md
    references/
      codex-tools.md       # Codex 工具映射（即上表内容）
```

### 10.4 Model 选择策略

不同 subagent 角色对模型能力的要求不同，应选择**能胜任该角色的最低成本模型**：

| Subagent 角色 | 推荐模型级别 | 理由 |
|---------------|-------------|------|
| Doc Scanner | 快速模型 | 结构化信息提取，不需要深度推理 |
| Spec Reviewer | 高能力模型 | 需要判断设计完整性和一致性 |
| Plan Reviewer | 高能力模型 | 需要评估计划合理性 |
| Implementer（简单 task） | 快速模型 | 1-2 个文件、spec 明确的机械实现 |
| Implementer（复杂 task） | 标准模型 | 多文件协调、需要理解上下文 |
| Spec Compliance Reviewer | 标准模型 | 对照 spec 检查实现 |
| Code Quality Reviewer | 高能力模型 | 需要架构判断力 |
| Fix Agent | 与原 task 同级或更高 | 修复需要至少同等理解力 |
| Doc Syncer | 快速模型 | 按模板填充，机械性任务 |

**模型级别定义**（具体模型名随平台和时间变化）：
- **快速模型**：低成本、高速度，适合机械任务（如 claude-sonnet、gpt-4o-mini）
- **标准模型**：平衡成本和能力（如 claude-sonnet 最新版、gpt-4o）
- **高能力模型**：最强推理能力（如 claude-opus、o1/o3）

**Task 复杂度信号**：
- 涉及 1-2 个文件 + spec 完整 → 快速模型
- 涉及多文件 + 需要集成协调 → 标准模型
- 需要设计判断 + 全局理解 → 高能力模型

---

## 11. 待决问题

以下事项在后续实现阶段需要进一步明确：

| 问题 | 说明 |
|------|------|
| Skills 文件存放位置 | `~/.claude/skills/`（全局）还是项目级别，待决定 |
