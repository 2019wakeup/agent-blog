# 第 6 篇 · Planning

> **搭建路线**：第 6 块积木——拼上"任务账本"：把目标拆成结构化步骤、每一步有状态和完成条件、计划未完成前不能草率收尾。
> **本篇目标**：理解三失败模式（配真实例子）；掌握"计划 = Harness 状态机"的转变；审批闸门与 Final Answer Guard 的工程形态。
> **难度**：⭐⭐⭐⭐

---

## 1. 为什么需要 Planner：长任务的三个失败模式

记忆能保存"我读到了什么"，但它不表达"我现在正在做哪一步"；上下文引擎能决定"哪些内容进入 request"，但它不判断"任务是否已完成"。把 Agent 放进真实长任务（如"阅读仓库最近一轮变更，对照前四章文章，写出完整第五章，并保存成 Markdown"），会出现三个典型失败模式（整合自 Tritium《Build An Agent From Scratch》[5]，CC BY-NC-SA）：

**失败模式一：任务漂移。** 执行几轮后，注意力被最近的工具结果牵着走，开始只处理眼前的信息，忘记最初目标——本来要写完整文章，读着读着变成只总结代码变更。

**失败模式二：假完成。** 模型生成了一个看起来很完整的 final answer，但其实漏掉了关键环节——还没核对 demo 命令、还没验证文件落盘，就宣布"第五章完成了"。模型不是撒谎，它只是没有一份可检查的清单。

**失败模式三：重复劳动。** 读过的文件、跑过的命令、确认过的结论没有被映射成结构化进度，于是过几轮又重新检查一遍。**它不是笨，只是没有一本可靠的"任务账本"**。

## 2. 前置回顾：ReAct 与 Reflexion

在拼 Planner 之前，回顾第 1 篇理论里提到的两种经典模式（它们现在有了用武之地）：

### ReAct（推理+行动交替）

```text
Question: 巴黎和伦敦哪个纬度高？

Thought 1: 我需要先知道两个城市的纬度，再比较。
Action 1: geocode(city="Paris")
Observation 1: 48.8566° N
Thought 2: 巴黎约 48.86°N，现在查伦敦。
Action 2: geocode(city="London")
Observation 2: 51.5074° N
Final: 伦敦（51.5°N）比巴黎（48.9°N）高。
```

Thought 留在上下文里帮模型保持连贯——**Planner 会给它加上"步骤状态"**：Thought 从"自言自语"变成"可检查的进度"。

### Reflexion（反思重试）

```python
for attempt in range(3):
    result = run_agent(task)
    if success(result): return result
    reflection = ask_model(f"上次失败：{result.error}，分析原因并给出修正方案")
    history.append(f"[反思] {reflection}")
```

**Planner 会把反思结果记录到步骤的证据里**：某步骤失败后，反思结论成为该步骤状态更新的依据。

## 3. 核心转变：计划是状态机，不是一段文字

很多人把 Plan-and-Solve 当成 prompt 技巧（先让模型输出计划，再按计划回答）。对单次问答有用，但对 Agent runtime 不够——如果计划只是一段 assistant message：

- 它可能被上下文引擎压缩成摘要（第 4 篇）——计划就"消失"了；
- 它没有机器可检查的步骤状态——无法判断"进行到哪了"；
- 它不能阻止模型提前 final answer——"假完成"防不住；
- 它无法表达"这个步骤完成的证据是什么"——无法验证；
- 它不能和工具权限、用户审批、会话持久化联动。

**计划的正确形态是 Harness 里的任务状态机**：

```text
User goal → 创建显式计划（Plan）
→ 审批（Review：计划创建后暂停，等用户批准再执行）
→ 每轮推进当前步骤（Solve：基于工具 observation 更新状态）
→ 计划不成立时显式修改（Replan）
→ 完成条件满足才停止
```

### 3.1 状态层（Planner 本体）

Planner 不依赖 LLM、不执行工具——它只做纯状态管理：

```ts
interface PlanState {
  objective: string              // 用户目标（不可变）
  steps: PlanStep[]              // 步骤列表
  currentStep: number            // 当前推进到哪一步
  revision: number               // 计划修订次数（Replan 时 +1）
  reviewStatus: "pending" | "approved" | "rejected"  // 审批状态
}

interface PlanStep {
  id: string
  description: string            // 这一步做什么
  status: "pending" | "in_progress" | "done" | "blocked"
  evidence?: string              // ★ 完成的证据（observation 摘要）
  completionCriteria: string     // ★ 完成条件（凭什么说完成）
}
```

**两个 ★ 字段是防"假完成"的关键**：`completionCriteria` 让"完成"有明确标准，`evidence` 让每一步的完成有据可查。

### 3.2 计划工具

允许模型创建、读取、修改、审批、清空计划（plan_create / plan_read / plan_update / plan_approve / plan_clear）——计划状态对模型是**可操作的对象**，不是只读文本。

### 3.3 动态计划快照

每轮把当前 plan 注入 request view（Prompter Builder 的一部分，第 4 篇①）——模型始终知道"我在哪一步、目标是什么"（状态锚定，第 1 篇 §5.2 机制 3）。

### 3.4 Final Answer Guard

计划仍有未完成步骤时，**阻止模型提前收尾**——宿主层面的硬约束，不依赖模型自觉：

```ts
function guardFinalAnswer(plan: PlanState): void {
  const open = plan.steps.filter(s => s.status !== "done")
  if (open.length > 0) {
    throw new PlanIncompleteError(
      `计划未完成：还有 ${open.length} 步未完成（${open.map(s => s.id).join(", ")}）`)
  }
}
```

## 4. 四层模型：Harness 各模块的分工

```text
chat history:    发生过什么            （事实记录）
memory:          哪些事实值得保留      （长期留存）
context engine:  本轮该看什么          （每轮视图）
planner:         目标是什么、当前做到哪一步、凭什么说完成了  （任务账本）
```

四个加起来，Agent 才从"会用工具的聊天窗口"变成"能完成长任务的系统"。**Planner 是最后一个拼上的 Harness 模块**——它把前几个模块的输出组织成"推进任务的行动依据"。

## 5. 审批闸门：计划未批准不能动手

计划创建后**暂停，等用户批准**再进入执行——尤其是写文件、跑命令这类破坏性操作。闸门的三重意义：

1. **防范围蔓延**：模型可能把任务理解歪（"修复登录 bug"变成"重构整个模块"）——计划先给人看，方向错了成本为零；
2. **防不可逆操作**：写文件、删数据一旦执行无法撤销——批准是最后一道人工防线；
3. **人机协同**：研究者拍板"方案对不对"，Agent 执行"怎么做"（第 9 篇 AutoResearcher 的立项原则正是如此："Agent 提方案，人拍板"）。

> 对照第 1 篇的"失败是第一等公民"：一个管"错怎么办"（错误回填），一个管"危险动作能不能做"（审批闸门）——两者互补。

## 6. DSH 参照：plan mode 与目标

| Planner 组件 | DSH 的对应 |
|---|---|
| 审批闸门 | **plan mode**：先只读探索、产出计划、exit_plan_mode 提交人工审批后才允许改文件；"用户口语上的同意不构成批准，只有 exit_plan_mode 走完评审才算" |
| 状态层/任务账本 | `goal`：持久化长任务目标，跨轮次续跑 |
| 动态计划快照 | 计划模式提示词段（每轮注入"你在计划模式"的规则） |

> 对照第 8 篇 §8.1：DSH 的 plan mode 覆盖了"Review 闸门"这一环；Tritium 系列展示完整四环（Plan→Review→Solve→Replan + Final Answer Guard）。两者合起来，就是 Plan-and-Solve 从思想到工程的全貌。

## 7. 练习与常见坑

> **练习 6.1**：给第 2 篇的循环加一个最小 Planner（状态对象 + plan_create/plan_update 工具 + 动态快照），跑"统计仓库代码行数并按模块分类"任务，观察任务漂移是否减少。
> **练习 6.2**：实现 Final Answer Guard（§3.4 的 guardFinalAnswer），故意让模型提前收尾，观察拦截效果。

**常见坑**：
1. **计划=prompt 技巧**：不建状态对象——压缩一来计划就没了；
2. **没有完成条件**：步骤"完成"全凭模型感觉——假完成防不住；
3. **审批走过场**：创建即批准——闸门失去意义；
4. **Guard 依赖模型自觉**：不加宿主层拦截——模型会"忘"。

## 8. 检验：读完本篇，你能回答吗

1. 长任务的三个失败模式是什么？"假完成"的根因是什么？（提示：没有可检查的清单）
2. 为什么计划不能只是一段 assistant message？（至少三条理由）
3. PlanStep 的两个 ★ 字段是什么？各防什么？
4. Planner 四组件是什么？Final Answer Guard 是宿主层还是模型层？
5. 审批闸门的三重意义是什么？
6. DSH 的 plan mode 覆盖了四环中的哪一环？

（答案都在上文：1→§1，2→§3，3→§3.1，4→§3，5→§5，6→§6）

## 本章小结

第 6 篇拼上了"任务账本"：计划是有状态的、可审批的、可防假完成的。到这一步，你的 Agent 已经能完成长任务。但它的"手"还是自己写的工具列表、"方法"还写在提示词里——下一篇拼上生态扩展。

**下一篇**：[第 7 篇 · 生态集成：MCP 与 Skills](07-生态集成-MCP与Skills.md) —— 扩展手与做事方法，顺便看清框架版图。
