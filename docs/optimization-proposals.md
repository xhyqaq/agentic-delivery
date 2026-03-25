# Agentic Delivery 性能优化方案

## 当前性能瓶颈

### 1. 串行评审流程
- **现状**: Task 1 (实现 → Spec Review → Code Review) → Task 2 (实现 → Spec Review → Code Review)
- **问题**: 完全串行，即使独立任务也无法并行评审
- **影响**: 10 个任务需要 30+ 次 subagent 调用（每任务 3 次）

### 2. Subagent 重启开销
- **现状**: 每次 review 都派发全新 subagent，无上下文复用
- **问题**: 每个 subagent 初始化有固定成本（模型加载、提示解析）
- **影响**: Review round 越多，重启成本越高

### 3. 两阶段强制串行
- **现状**: Spec Review MUST完成 → 才能 Code Quality Review
- **问题**: 即使 Spec 简单/明确，也无法跳过或并行
- **影响**: 最少 2 次 subagent 调用/任务

### 4. 上下文重建成本
- **现状**: 每个 reviewer 都要重新读取 git diff 和相关文件
- **问题**: 相同内容多次传输（Spec reviewer 看一遍，Code reviewer 又看一遍）
- **影响**: Token 使用量翻倍，API 调用时间翻倍

## 优化策略（按优先级排序）

---

## 🚀 优化 1: 批量并行评审（高影响）

### 当前流程
```
Task 1 实现 → Task 1 Spec Review → Task 1 Code Review →
Task 2 实现 → Task 2 Spec Review → Task 2 Code Review →
Task 3 实现 → Task 3 Spec Review → Task 3 Code Review
```

### 优化后流程
```
并行批次 1: Task 1,2,3 实现（并行）
  ↓
并行批次 2: Task 1,2,3 Spec Review（并行）
  ↓
并行批次 3: Task 1,2,3 Code Review（并行）
```

### 实现方案

**修改 Stage 5-6 执行逻辑:**

```markdown
### 新策略: 分阶段批量执行

**Phase 1: Parallel Implementation**
- 识别所有可并行的任务（无文件冲突）
- 同时派发所有 implementer subagents
- 等待所有实现完成，收集 checkpoint commits

**Phase 2: Parallel Spec Review**
- 对所有 DONE 的任务，同时派发 Spec Reviewers
- 并行检查所有任务的 Spec Compliance
- 收集需要修复的任务列表

**Phase 3: Batch Fix (if needed)**
- 对需要修复的任务，派发 Fix subagents
- 修复完成后，重新进入 Phase 2（最多 3 轮）

**Phase 4: Parallel Code Quality Review**
- 对所有通过 Spec Review 的任务，同时派发 Code Quality Reviewers
- 并行检查代码质量
- 收集需要修复的任务列表

**Phase 5: Batch Fix (if needed)**
- 对需要修复的任务，派发 Fix subagents
- 修复完成后，重新进入 Phase 4（最多 3 轮）

**Phase 6: Batch Doc Sync**
- 对所有通过的任务，同时派发 doc-syncer subagents
- 并行更新 changelog
```

### 性能提升
- **10 个独立任务**: 30 次串行调用 → 6 批次并行（假设无修复）
- **预估加速**: 3-5x（取决于任务独立性）

---

## 🔥 优化 2: 智能评审路由（中影响）

### 动态选择评审策略

不是所有任务都需要两阶段评审：

**快速路径（Fast Track）** - 适用条件：
- 单文件修改 < 50 行
- 无新增 API/接口
- 无架构变更
- 是纯函数/工具函数

→ **只做 Code Quality Review**（跳过 Spec Review）

**标准路径（Standard Track）** - 适用条件：
- 多文件修改
- 有新增 API
- 有业务逻辑变更

→ **两阶段评审**（现有流程）

**重度路径（Heavy Track）** - 适用条件：
- 跨域集成（frontend + backend）
- 架构变更
- 安全敏感代码

→ **三阶段评审**（Spec → Code → Integration）

### 实现方案

在 `review-fix-strategy.md` 中添加路由决策树：

```markdown
## Review Track Selection (Stage 5 开始前)

Main agent 分析每个任务，打上标签：

| 任务特征 | Track | Review 流程 |
|---------|-------|------------|
| 单文件 < 50 行，纯逻辑 | Fast | Code Quality Only |
| 多文件，业务逻辑 | Standard | Spec → Code Quality |
| 跨域集成，架构变更 | Heavy | Spec → Code → Integration |
```

### 性能提升
- **Fast Track 任务**: 减少 33% 评审时间（1 次 review vs 2 次）
- **预估**: 30-50% 任务可走 Fast Track → 整体加速 15-20%

---

## ⚡ 优化 3: 合并式评审（Merged Review）（高影响）

### 核心思想
将 Spec Compliance 和 Code Quality 合并为单次 review，但输出分为两个维度

### 当前问题
- Spec Reviewer 读 git diff → 生成报告
- Code Quality Reviewer 又读同一个 git diff → 生成报告
- **重复劳动**: 两个 reviewer 看同样的代码

### 优化方案

**单一 Unified Reviewer**，输出结构化报告：

```json
{
  "spec_compliance": {
    "status": "pass" | "fail",
    "missing_requirements": [...],
    "extra_implementation": [...]
  },
  "code_quality": {
    "status": "pass" | "fail",
    "issues": [
      {"severity": "Critical|Important|Minor", "description": "..."}
    ],
    "strengths": [...]
  },
  "recommendation": "approve" | "fix_spec_first" | "fix_quality" | "fix_both"
}
```

### 修复策略
- **只有 Spec 问题**: Fix subagent 修复 → 重新 Unified Review
- **只有 Quality 问题**: Fix subagent 修复 → 重新 Unified Review
- **两者都有**: Fix subagent 修复 → 重新 Unified Review

### 性能提升
- **每个任务**: 2 次 review → 1 次 review
- **减少 Token 使用**: git diff 只传输 1 次（而非 2 次）
- **预估加速**: 40-50%（最大收益优化）

---

## 🧠 优化 4: 上下文缓存与复用（中影响）

### 当前浪费
每个 subagent 都接收完整的：
- 任务全文（copy-paste from plan）
- Spec 片段
- Git diff
- 项目约定

### 优化方案

**引入共享上下文层**:

1. **任务开始时**，main agent 构建 "Review Context Package":
   ```markdown
   ## Review Context for Task N
   - Task description: [...]
   - Relevant spec: [...]
   - Changed files: [file1.ts, file2.ts]
   - Git diff (BASE_SHA..HEAD_SHA): [cached]
   - Project conventions: [cached]
   ```

2. **存储为临时文件**: `docs/<project>/<feature>/.review-cache/task-N-context.md`

3. **Reviewer 接收**: 文件路径而非全文（减少 prompt 大小）

### 性能提升
- **减少 prompt 大小**: 每个 reviewer 节省 500-1000 tokens
- **减少重复传输**: git diff 只构建 1 次
- **预估加速**: 10-15%

---

## 🎯 优化 5: 自适应 Review Rounds（低影响）

### 当前固定策略
- 所有任务都允许最多 3 轮 review
- 即使第 1 轮就完美通过，也预留了 3 轮的"预算"

### 优化方案

**动态调整 max rounds**:

| 任务复杂度 | Max Rounds |
|-----------|-----------|
| Simple (< 50 lines, 单文件) | 1 round |
| Medium (50-200 lines, 2-3 文件) | 2 rounds |
| Complex (> 200 lines, 多文件集成) | 3 rounds |

**提前终止策略**:
- 如果连续 2 个任务都 1 次通过 → 后续任务降低警戒（信任 implementer）
- 如果任务失败 2 轮 → 提前 escalate to user（而非等 3 轮）

### 性能提升
- **减少不必要的重试**: 简单任务失败立即 escalate
- **预估加速**: 5-10%（边际收益）

---

## 🛠️ 优化 6: Implementer 质量前置（高影响，但需改动 subagent）

### 核心思想
让 Implementer 的 self-review 更严格，减少后续 review 失败率

### 当前问题
- Implementer 的 self-review 是"自觉"的（prompt 提示）
- 没有强制检查清单
- Reviewer 发现的问题，Implementer 其实可以自己避免

### 优化方案

**在 implementer-prompt.md 中添加强制 Pre-submission Checklist**:

```markdown
## Before Reporting DONE

YOU MUST complete this checklist (report results in your response):

**Spec Compliance Self-Check:**
- [ ] All required features implemented (list each one)
- [ ] No extra features added beyond spec
- [ ] All edge cases from spec handled

**Code Quality Self-Check:**
- [ ] No magic numbers (extract to constants)
- [ ] No TODO/FIXME comments left
- [ ] No console.log/debug code
- [ ] Consistent naming with project conventions
- [ ] All new functions have clear purpose

**Test Coverage Self-Check:**
- [ ] All core paths tested
- [ ] Edge cases tested
- [ ] Tests actually pass (not just written)

**Report format:**
✅ = passed, ⚠️ = concern, ❌ = failed
```

### 配合 Reviewer 调整

Reviewer 的工作变为"验证 Implementer 的 self-check"：
- 如果 Implementer 漏报问题 → Critical issue（说明 self-review 失效）
- 如果 Implementer 正确识别问题 → 信任度提升

### 性能提升
- **减少 review 失败率**: 从 30% → 10%（假设）
- **减少 fix rounds**: 平均从 1.5 轮 → 0.3 轮
- **预估加速**: 20-30%（长期收益）

---

## 📊 综合优化方案对比

| 优化策略 | 实现难度 | 性能提升 | 架构影响 | 优先级 | 状态 |
|---------|---------|---------|---------|-------|------|
| **优化 1: 批量并行评审** | 🔶 中等 | 🚀 3-5x | 🟡 中（需改 Stage 5-6 逻辑） | ⭐⭐⭐⭐⭐ P0 | ⏳ TODO |
| **优化 3: 合并式评审** | 🔷 中等 | 🔥 40-50% | 🟡 中（需改 reviewer prompt） | ⭐⭐⭐⭐⭐ P0 | ⏳ TODO |
| **优化 6: Implementer 质量前置** | 🟢 简单 | 🎯 20-30% | 🟢 低（只改 prompt） | ⭐⭐⭐⭐ P1 | ⏳ TODO |
| **优化 2: 智能评审路由** | 🔶 中等 | ⚡ 15-20% | 🟡 中（需添加决策逻辑） | ⭐⭐⭐ P2 | ✅ DONE |
| **优化 4: 上下文缓存** | 🔷 中等 | 💾 10-15% | 🟢 低（添加缓存层） | ⭐⭐ P3 | ⏳ TODO |
| **优化 5: 自适应 Rounds** | 🟢 简单 | 📈 5-10% | 🟢 低（只改配置） | ⭐ P4 | ⏳ TODO |

---

## 🎬 实施路线图

### Phase 1: 快速收益（1-2 天）
1. ✅ **优化 6**: 增强 Implementer self-check（改 1 个 prompt）
2. ✅ **优化 5**: 自适应 review rounds（改配置）

**预期**: 25-40% 加速，零架构风险

---

### Phase 2: 核心重构（3-5 天）
3. ✅ **优化 2**: 智能评审路由 ✅ **COMPLETED - 2026-03-25**
   - 添加 Review Track Selection (Fast/Standard/Heavy)
   - 修改 Stage 5 逻辑（5A: Track Selection, 5B: Implementation）
   - 更新 review-fix-strategy skill 包含 Track Selection 决策树
   - 更新 subagent-driven-development skill 集成 track routing
   - Track-specific max rounds: Fast=1, Standard=2, Heavy=3

4. ✅ **优化 3**: 合并式评审
   - 设计 Unified Reviewer prompt
   - 修改 Stage 6 逻辑（review 调用链）
   - 保持向后兼容（可通过 flag 切换）

5. ✅ **优化 1**: 批量并行评审
   - 重构 Stage 5-6 为 phase-based execution
   - 添加并行 dispatch 协调逻辑
   - 处理批量失败场景

**预期**: 累计 4-6x 加速，需要充分测试

---

### Phase 3: 精细化优化（1-2 天）
6. ✅ **优化 4**: 上下文缓存
   - 实现 Review Context Package 生成
   - 修改 reviewer prompts 使用缓存

**预期**: 累计 5-8x 加速，完整优化

---

## 🧪 性能基准测试计划

### 测试场景
**Scenario A: 小型功能（3 个独立任务）**
- 现状: ~15 分钟
- 目标: <3 分钟（5x 加速）

**Scenario B: 中型功能（10 个任务，5 并行 + 5 串行）**
- 现状: ~45 分钟
- 目标: <10 分钟（4.5x 加速）

**Scenario C: 大型功能（20 个任务，frontend + backend）**
- 现状: ~90 分钟
- 目标: <15 分钟（6x 加速）

### 测试指标
- Total execution time（端到端时间）
- Subagent dispatch count（调用次数）
- Token usage（API 成本）
- Review failure rate（质量指标，确保优化不损害质量）

---

## 🔒 质量保障

优化后必须保持：
- ✅ Zero false positives（不通过的不能通过）
- ✅ Zero false negatives（该发现的问题必须发现）
- ✅ Same review depth（评审深度不降低）

**防护措施**:
1. 并行评审时，每个 reviewer 仍然是独立的 subagent（无交叉污染）
2. 合并式评审仍然输出独立的 spec/quality 判断（可追溯）
3. Fast Track 只用于明确的简单场景（有清晰规则）

---

## 💡 其他探索方向（未来）

### 7. Progressive Review（渐进式评审）
- Implementer 每完成一个文件 → 立即轻量级 review
- 而非等所有文件完成后一次性 review
- **好处**: 提前发现方向性错误

### 8. Review Result Caching
- 如果代码未变，复用上次 review 结果
- 适用于跨 session 的增量开发

### 9. Model Tier Auto-Selection
- 根据任务历史成功率，动态选择模型
- 简单任务用 Fast model，复杂任务自动升级到 Capable

### 10. Parallel Implementer Coordination
- 当两个并行任务操作相邻代码时，自动协调接口
- 避免后期集成问题

---

## 📞 反馈与调整

优化是迭代的过程。实施后请关注：
- 用户感知速度（主观）
- 实际指标（客观）
- 质量回归（防护）

根据反馈调整策略优先级。
