# 第 2 篇 · 最小 Agent Loop

> **搭建路线**：第 2 块积木——拼上能跑的最小循环：模型会思考、工具能执行、结果能回填、循环能继续、答案能终止。
> **本篇目标**：实现并吃透 35 行核心循环；掌握函数调用的四个工程细节；理解"最小实现 → 工程实现"的五个扩展点（每个都配代码）。
> **难度**：⭐⭐

---

## 1. 函数调用的四个工程细节

第 1 篇讲了函数调用的三阶段机制。动手实现前，补齐四个工程细节（都是从踩坑中来的）：

### 1.1 arguments 是 JSON 字符串

模型返回的 `arguments` 是字符串不是对象，必须 `json.loads` 解析——而且要处理解析失败：

```python
import json
try:
    args = json.loads(call.function.arguments)
except json.JSONDecodeError as e:
    # 模型可能给非法 JSON：把错误回填给模型，让它重来
    messages.append({"role": "tool", "tool_call_id": call.id,
                     "content": f"ERROR: 参数解析失败 {e}，请重新给出合法 JSON"})
```

### 1.2 一次响应可能多个 tool_calls

模型可以一次请求多个工具（并行工具调用）。执行要支持并发，但**写回历史的顺序必须与请求顺序一致**：

```python
from concurrent.futures import ThreadPoolExecutor
if msg.tool_calls:
    with ThreadPoolExecutor(max_workers=8) as pool:
        results = list(pool.map(execute_call, msg.tool_calls))  # 顺序与输入一致
    for call_id, result in results:
        messages.append({"role": "tool", "tool_call_id": call_id, "content": result})
```

> **铁律**："性能可以并行，模型看到的上下文不能乱。" 只读调用可并发，写操作串行——生产框架的调度契约（第 8 篇 PTC 的实现）。

### 1.3 失败的正道：把错误回填给模型

工具失败太常见了：文件不存在、命令非零退出、网络 429、参数 schema 不匹配、权限不足。**失败如果不进上下文，模型就没有自我修正的机会**：

```python
try:
    result = run_shell(**args)
except Exception as e:
    result = f"ERROR: {type(e).__name__}: {e}"   # 作为正常 tool 消息回填
# 模型看到 ERROR 后通常会换参数重试或说明原因
```

### 1.4 描述是模型唯一的认知来源

工具描述的质量直接决定调用质量：描述第一句要点题（"查询指定城市的当前天气，当用户问天气时使用"优于"天气工具"）；工具数量建议 ≤ 20 个（多了模型"选择困难"甚至忽略工具直接瞎答）。

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
1. **信号灯**：`tool_calls is None` 决定循环是否继续——对应第 1 篇的 `finish_reason`；
2. **契约**：工具只暴露 name/description/parameters 给模型，实现是黑盒；
3. **回填闭环**：assistant 消息（含 tool_calls）和 tool 消息（含 call_id）必须都进历史——缺任何一条，模型就"失忆"；
4. **截断**：`str(result)[:3000]`——上下文保护从第一行代码就开始。

### 一次真实运行 trace

```text
[step 0] 调用 run_shell({'cmd': 'find . -name "*.py" | wc -l'})
[step 1] 调用 read_file({'p': 'main.py'})
[step 2] 最终回答: 当前目录共有 42 个 .py 文件，总行数统计完成。
```

注意：每一步都是一次完整的 API 往返。5 步任务 = 5 次往返——这就是第 8 篇 PTC 模式要解决的问题。

> **练习 2.1**：跑通后回答三个问题：① 为什么 content 要截断？② 工具抛异常会发生什么？（提示：循环崩溃——生产做法见 §1.3）③ 把任务换成需要 5 步以上的，观察循环如何协作。

## 3. 五个扩展点：从"能跑的 demo"到"能生产的系统"

真实工程（参考 Codex 与 Pi 的生产实现）在最小循环上预留五个扩展点（整合自 Tritium《Build An Agent From Scratch》[2]，CC BY-NC-SA）：

### 扩展点 1：流式与非流式共用同一个 loop

Agent Loop 只等待最终 AssistantMessage；流式只是 LLM Client 的增强能力：

```ts
if (!isStreamingLlmClient(this.llm)) {
  return this.llm.complete(request)
}
for await (const event of this.llm.stream(request)) {
  // thinking_delta / text_delta / tool_call_delta / done
}
// loop 本体不感知流式与否，只等一个最终消息
```

### 扩展点 2：工具并行执行，但历史顺序稳定

```ts
const results = await Promise.all(
  toolCalls.map(async (tc) => this.executor.execute(tc, signal))
)   // Promise.all 结果顺序与输入数组一致 → 写回 history 的顺序稳定
```

### 扩展点 3：工具错误不打断循环

```ts
// 错误被转换为带 isError 标记的 tool 消息，模型"读到失败"再纠正
{ role: "tool", toolCallId: tc.id, toolName: tc.name,
  content: "...error message...", isError: true }
```

### 扩展点 4：事件系统先行

agent_start / turn_start / message / tool_start / tool_end / agent_end 成为一等接口——CLI、Web UI、日志系统订阅同一条事件流，后续加可观测性不用拆开重写循环：

```ts
const agent = new Agent({ onEvent(event) { /* log / UI / SSE */ } })
// 或：for await (const event of agent.runEvents(input)) { ... }
```

### 扩展点 5：模型配置隔离在 LLM 边界

```ts
await agent.run("...", { reasoning: { effort: "high", summary: "concise" } })
// Agent 只传通用配置；具体 provider 字段由 LlmClient 适配层映射
```

> 五条不是优化建议，是"从循环到系统"的工程骨架。第 8 篇会看到 DSH 的调度、事件流、ToolCallError 正是同一套思想的工程化。

## 4. DSH 参照：生产级的 Agent Loop 长什么样

你刚实现的循环，在 DSH 里有三个对应物（细节在第 8 篇，这里先建立映射）：

| 你的实现 | DSH 的对应 | 生产级差异 |
|---|---|---|
| `client.chat.completions.create` | agent loop + LLM route（宿主平面） | 模型路由与多 Provider 由宿主组合管理 |
| `eval(tool)(args)` 手动执行 | 工具注册表 + 完整工具管线 | 每次调用走 pre-execute → guards → execute → post-execute → result，失败以 `ToolCallError`（只含 toolName + message）抛回 |
| 手动 `messages.append` 回填 | 事件流 + 会话投影 | 每个子调用有 `tool/code-dispatch` 事件，UI/审计可回放 |
| `[:3000]` 截断 | 上下文引擎（第 4 篇） | 六层流水线：预算/压缩/Handoff/动态折叠 |

> **关键认知**：DSH 的 PTC 模式（第 8 篇）把"35 行循环的每轮往返"压缩为"一次 run_code 程序执行"——但循环语义不变：模型请求动作、宿主执行、观察回填。**思想不变，工程形态变**。

## 5. 常见坑（本系列读者的高频踩坑）

1. **忘了把 assistant 消息（含 tool_calls）加进历史**——模型第二轮就"失忆"，反复要同一个工具；
2. **tool_call_id 不匹配**——tool 消息回填找不到对应请求，API 报错；
3. **不给错误留上下文**——工具抛异常直接 break，模型完全不知道发生了什么；
4. **一次塞太多工具**——模型"选择困难"，甚至忽略工具直接瞎答（第 1 篇 §3.3）。

## 6. 检验：读完本篇，你能回答吗

1. 函数调用四个工程细节是什么？为什么"错误回填给模型"是正道？
2. 35 行循环的"信号灯"是什么？assistant 消息和 tool 消息为什么都必须进历史？
3. 工具并行的铁律是什么？（提示：性能可以并行，什么不能乱？）
4. 五个扩展点里，哪个对应"事件流先行"？工程意义是什么？
5. 你的最小循环中，哪一处是"上下文保护"？（提示：第一行代码就开始）
6. 常见坑第一条是什么？怎么避免？

（答案都在上文：1→§1，2→§2，3→§1.2，4→§3，5→§2，6→§5）

## 本章小结

第 2 篇拼上了第一块能跑的积木：循环能转、工具能执行、结果能回填、错误能纠正。但你的工具只有两个教学函数，还碰不了真实世界——下一篇拼上"手脚"。

**下一篇**：[第 3 篇 · 工具系统与基础堆量](03-工具系统与基础堆量.md) —— 文件/Shell/Web 工具、Provider 边界、截断防线。
