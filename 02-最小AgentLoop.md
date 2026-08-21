# 第 2 篇 · 最小 Agent Loop

> **搭建路线**：第 2 块积木——拼上能跑的最小循环：模型会思考、工具能执行、结果能回填、循环能继续、答案能终止。
> **本篇目标**：实现 35 行核心循环；理解函数调用的工程细节；掌握"最小实现 → 工程实现"的五个扩展点。
> **难度**：⭐⭐

---

## 1. 函数调用的工程细节

第 1 篇讲了函数调用的三阶段机制。动手实现前，补齐四个工程细节（都是从踩坑中来的）：

1. **arguments 是 JSON 字符串**：必须 `json.loads` 解析，且要处理解析失败（模型可能给非法 JSON）；
2. **一次响应可能多个 tool_calls**：循环要支持并行执行——**只读调用可并发，写操作串行**（生产框架的调度契约）；
3. **失败的正道是"把错误回填给模型"**：把异常转成 `isError: true` 的 tool 消息回填，让模型读到失败并自我修正——失败不进上下文，模型就没有纠正机会（文件不存在、命令非零退出、网络 429、schema 不匹配……真实世界里工具失败是常态）；
4. **工具描述是模型唯一的认知来源**：描述第一句要点题（"查询指定城市的当前天气，当用户问天气时使用"优于"天气工具"），工具数量建议 ≤ 20 个。

## 2. 最小 Agent Loop：35 行核心

把第 1 篇的循环图写成代码（刻意省略工具实现细节，让你把注意力放在唯一的重点——循环本身）：

```python
import json
from openai import OpenAI
client = OpenAI(base_url="https://api.deepseek.com", api_key="sk-...")

# ── 工具层：实现细节不重要，重要的是"契约"──
def run_shell(cmd: str) -> str:  return "总行数: 42"
def read_file(p: str) -> str:    return "file content"

TOOLS = [{"type": "function", "function": {"name": "run_shell",
    "description": "执行 shell 命令返回 stdout",
    "parameters": {"type": "object", "properties": {"cmd": {"type": "string"}},
                   "required": ["cmd"], "additionalProperties": False}}}]

# ── Agent 循环：全部"智能"发生的地方 ──
def agent(task: str, max_steps: int = 10):
    messages = [{"role": "user", "content": task}]
    for _ in range(max_steps):
        msg = client.chat.completions.create(
            model="deepseek-chat", messages=messages, tools=TOOLS
        ).choices[0].message
        messages.append({"role": "assistant", "content": msg.content or "",
                         "tool_calls": msg.tool_calls})
        if not msg.tool_calls:               # 信号灯：没有工具调用 = 最终回答
            return msg.content
        for call in msg.tool_calls:          # 执行 + 回填（错误也回填，让模型纠正）
            result = eval(call.function.name)(**json.loads(call.function.arguments))
            messages.append({"role": "tool", "tool_call_id": call.id,
                             "content": str(result)[:3000]})

print(agent("统计当前目录 .py 文件总行数"))
```

**读懂这 35 行的四个层次**（比代码本身更重要）：
1. **信号灯**：`tool_calls is None` 决定循环是否继续；
2. **契约**：工具只暴露 name/description/parameters 给模型，实现是黑盒；
3. **回填闭环**：assistant 消息（含 tool_calls）和 tool 消息（含 call_id）必须都进历史；
4. **截断**：`str(result)[:3000]`——上下文保护从第一行代码就开始。

> **练习 2.1**：跑通后回答三个问题：① 为什么 content 要截断？② 工具抛异常会发生什么？（提示：循环崩溃——生产做法见下节"错误进循环"）③ 5 步任务需要几次往返？（提示：这就是第 8 篇 PTC 模式要解决的问题）

## 3. 五个扩展点：从"能跑的 demo"到"能生产的系统"

真实工程（参考 Codex 与 Pi 的生产实现）在最小循环上预留五个扩展点——它们决定了 demo 与系统的距离（整合自 Tritium《Build An Agent From Scratch》[2]，CC BY-NC-SA）：

1. **流式与非流式共用同一个 loop**：Agent Loop 只等待最终 AssistantMessage；流式（thinking_delta/text_delta/tool_call_delta）只是 LLM Client 的增强能力；
2. **工具可并行执行，但历史顺序必须稳定**：并发跑（Promise.all），但写回 history 的顺序与请求顺序一致——"性能可以并行，模型看到的上下文不能乱"；
3. **工具错误不打断循环**：任何失败转换成带 isError 标记的 tool 消息回填（见 §1 第 3 条）；
4. **事件系统先行**：agent_start / turn_start / message / tool_start / tool_end / agent_end 成为一等接口，CLI、Web UI、日志系统订阅同一条事件流——后续加可观测性不用拆开重写循环；
5. **模型配置隔离在 LLM 边界**：reasoning effort、模型名等通用配置由 Agent 传给 LLM Client，由 Client 映射成具体 provider 的字段。

> 五条不是优化建议，是"从循环到系统"的工程骨架。第 8 篇会看到 DSH 的调度、事件流、ToolCallError 正是同一套思想的工程化。

## 4. DSH 参照：生产级的 Agent Loop 长什么样

你刚实现的循环，在 DSH 里有三个对应物（细节在第 8 篇，这里先建立映射）：

| 你的实现 | DSH 的对应 | 生产级差异 |
|---|---|---|
| `client.chat.completions.create` | agent loop + LLM route（宿主平面） | 模型路由与多 Provider 由宿主组合管理 |
| `eval(tool)(args)` 手动执行 | 工具注册表 + 完整工具管线 | 每次调用走 pre-execute → guards → execute → post-execute → result，失败以 `ToolCallError`（只含 toolName + message）抛回 |
| 手动 `messages.append` 回填 | 事件流 + 会话投影 | 每个子调用有 `tool/code-dispatch` 事件，UI/审计可回放 |
| `[:3000]` 截断 | 上下文引擎（第 4 篇将实现） | 六层流水线：预算/压缩/Handoff/动态折叠 |

> **关键认知**：DSH 的 PTC 模式（第 8 篇）把"35 行循环的每轮往返"压缩为"一次 run_code 程序执行"——但循环语义不变：模型请求动作、宿主执行、观察回填。**思想不变，工程形态变**。

## 5. 检验：读完本篇，你能回答吗

1. 函数调用四个工程细节是什么？为什么"错误回填给模型"是正道？
2. 35 行循环的"信号灯"是什么？assistant 消息和 tool 消息为什么都必须进历史？
3. 五个扩展点里，哪个对应"工具错误不打断循环"？哪个对应"性能可以并行，上下文不能乱"？
4. 你的最小循环中，哪一处是"上下文保护"？（提示：第一行代码就开始）
5. DSH 的 ToolCallError 与"错误回填"是什么关系？

## 本章小结

第 2 篇拼上了第一块能跑的积木：循环能转、工具能执行、结果能回填、错误能纠正。但你的工具只有两个教学函数，还碰不了真实世界——下一篇拼上"手脚"。

**下一篇**：[第 3 篇 · 工具系统与基础堆量](03-工具系统与基础堆量.md) —— 文件/Shell/Web 工具、Provider 边界、截断防线。
