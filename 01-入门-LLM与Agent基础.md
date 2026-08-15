# 第 1 篇 · LLM 与 Agent 基础

> **难度**：⭐⭐（会写 Python/JS 即可）
> **本篇目标**：把 Agent 的底层机制**讲透**——不只是"怎么调 API"，而是"为什么模型会这样做、上下文为什么有限、工具调用为什么能工作"。代码只保留最能说明原理的核心片段。

---

## 1. 先搞懂 LLM 在做什么

### 1.1 Token 化：模型不是"读字"，是"读符号"

大语言模型（LLM）的输入输出单位不是字符，而是 **token**——文本被分词器切成的小块。主流模型使用 **BPE（字节对编码）**：从单字符开始，反复把"出现频率最高的相邻字符对"合并成一个新符号，直到达到预设词表大小。

```text
原始文本: h e l l o   w o r l d
合并高频对 → [hello] [ world]    ← 常见词/子词成为一个 token
```

三个直接后果，每一个都影响你的工程决策：

1. **计费按 token**：输入+输出 token 数 × 单价 = 成本（第 4 篇的成本控制全靠它）；
2. **上下文上限是 token 数**：128K 上下文 ≈ 8~10 万汉字，不是字符数；
3. **KV 缓存按 token 复用**：前缀 token 序列不变才能命中缓存（见 1.3）。

> 中文大约 1~2 个汉字 = 1 个 token；代码的缩进、常见关键字是高频对，会形成专用 token——这解释了为什么代码类 token 密度高、模型擅长代码。

### 1.2 生成过程：为什么模型是"概率分布"而不是"查字典"

LLM 的核心是一个条件概率分布：给定前面所有 token，预测下一个 token 的概率。**关键认知：这不是检索，是采样**——模型输出的每个字都是从分布中抽出来的：

```text
P(next | "今天天气") → { "真": 0.4, "很": 0.3, "不错": 0.2, ... }
采样 → "很"
P(next | "今天天气很") → { "好": 0.5, ... } → "好"
输出："今天天气很好"
```

两个参数决定了这个分布的"形状"：

| 参数 | 作用 | 直觉 |
|---|---|---|
| `temperature` | 分布尖锐度（0~1） | 越低越"只选最可能"，越高越"敢于冒险" |
| `top_p` | 只从累积概率 top_p 的 token 里采样 | 砍掉长尾低概率 token，防止"胡说" |

> **为什么这决定工程行为**：写代码用低 temperature（0~0.3，确定性优先）；评测必须固定 temperature=0（可复现，第 4 篇评测的硬要求）。**"模型会随机"不是 bug，是机制**——这也是为什么 Agent 需要重试、反思和评测。

### 1.3 上下文窗口与 KV 缓存：Agent 工程的物理定律


**一个直觉类比**：注意力机制像**手电筒在黑暗房间里找东西**——内容越多（房间越大），手电筒的光束越分散，每件物品被照到的注意力越弱。这就是为什么上下文越长，模型越“记不住”早期内容：不是它“忘了”，是注意力被摊薄了。

**上下文窗口** = 模型一次推理能"看到"的 token 上限。它为什么存在？因为模型内部是**注意力机制**：每个 token 都要与前面所有 token 计算相关性，上下文越长，计算量越大、信息越被"稀释"（注意力被摊薄）。这是**物理限制**，不是工程偷懒。

**KV 缓存**是推理引擎的优化：生成时每个 token 的注意力中间状态（Key/Value）被缓存，**只要请求前缀的 token 序列不变，前缀状态直接复用**：

```text
请求 A: [system][工具描述][历史] [新问题]   ← 前缀被缓存
请求 B: [system][工具描述][历史] [新问题']  ← 前缀相同 → 命中缓存（省钱省时）
请求 C: [system][工具描述改一个字][历史]... ← 前缀变了 → 缓存全部失效
```

**KV 缓存类比**：像**备菜**——请求 A 时已经把“system + 工具描述 + 历史”全部切好洗净（算好 Key/Value）；请求 B 只是加了句新问题，直接下锅（复用缓存）；但如果你改动前缀的任何一个字，等于把已备好的菜倒掉重新切（缓存全失效）。所以生产框架把提示词前缀当作“神圣不可侵犯”的区域。

> **工程启示（生产框架都在为它优化）**：提示词前缀要**稳定**——工具描述顺序固定、文本确定、历史只追加不修改。DSH 的 SDK 生成"字典序、逐字节确定"就是为了这个（第 3 篇详述）。

### 1.4 一次真实的 API 调用

用 OpenAI SDK 调 DeepSeek 模型（协议兼容）：

```python
# pip install openai
from openai import OpenAI
client = OpenAI(api_key="sk-...", base_url="https://api.deepseek.com")

resp = client.chat.completions.create(
    model="deepseek-chat",
    messages=[{"role": "user", "content": "1 + 1 = ?"}],
    temperature=0.2,
)
print(resp.choices[0].message.content)   # 答案
print(resp.usage)                        # token 统计（计费依据）
```

注意响应里的 `finish_reason` 字段——它只有几个取值（`stop` / `length` / `tool_calls` / `content_filter`），**它就是 Agent 循环的"信号灯"**：下一节会看到，模型决定调用工具时，这个字段会变成 `tool_calls`。

> **练习 1.1**：跑通上面代码，打印完整响应 JSON，观察 `finish_reason` 与 `usage`。

## 2. 什么是 Agent？

> **Agent（智能体）** = LLM（大脑）+ 工具（手脚）+ 循环（编排）

拆开看三个词的含义：

- **大脑**：LLM 负责理解、推理、决策——但它**只会输出文本**；
- **手脚**：工具是你注册给模型的可调用函数——执行代码、读写文件、搜索网页；
- **编排**：循环代码把两者串起来——这是 99% 的"智能"实际发生的地方，也是你（开发者）唯一完全掌控的部分。

Agent 循环的核心只有四步：

```text
┌────────────────────────────────────────────┐
│  while 任务未完成:                          │
│    1. 打包"任务 + 历史 + 工具清单"成提示词    │
│    2. 调 LLM，得到下一步（文本 或 工具请求）  │
│    3. 若是工具请求：执行，结果放回上下文      │
│    4. 若是最终回答：输出，结束              │
└────────────────────────────────────────────┘
```

**最重要的心智模型**：模型本身不执行任何东西。它只输出两种东西之一——普通文本（`finish_reason: stop`，最终回答）或工具调用请求（`finish_reason: tool_calls`）。执行永远发生在你的代码里。这个认知贯穿全部后续内容。

> 一个真实回合的 trace（"把 123 和 456 相加再乘 2"）：第 1 次请求 → 模型要 `add(123,456)` → 你执行得 579 → 回填 → 第 2 次请求 → 模型要 `multiply(579,2)` → 回填 → 第 3 次请求 → 输出 1158。**三次完整往返**——这就是为什么多步任务慢，也是 PTC 模式（第 3 篇）存在的理由。

## 3. 函数调用（Function Calling）：Agent 的地基

### 3.1 本质：工具调用也是"文本生成"

先建立一个反直觉但关键的认知：**模型请求调用工具，本质上仍然是生成了一段文本**——只是这段文本恰好符合我们定义的 JSON 格式。模型看到工具描述（一段文本），基于任务生成"我想调 get_weather，参数是 {city: 北京}"（一段 JSON 文本）。**模型没有执行任何东西，也感知不到工具的"真实"存在**。

这个认知解释了两个现象：
- **模型会"幻觉"参数**（编造不存在的城市名）——因为它只是按概率续写文本；
- **工具描述的质量直接决定调用质量**——描述是模型对工具唯一的认知来源。

### 3.2 三阶段机制

**阶段一：告诉模型"你有什么工具"**——每个工具是一个 JSON Schema 描述（模型看到的是这段文本）：

```python
tools = [{
    "type": "function",
    "function": {
        "name": "get_weather",
        "description": "查询指定城市的当前天气",   # ← 模型靠它做决策
        "parameters": {
            "type": "object",
            "properties": {"city": {"type": "string"}},
            "required": ["city"],
            "additionalProperties": False,          # 防止发明字段
        },
    },
}]
```

**阶段二：模型返回调用请求**——注意 `finish_reason: "tool_calls"` 与 `arguments` 是 **JSON 字符串**（必须 parse 才能用）：

```json
{
  "choices": [{
    "finish_reason": "tool_calls",
    "message": {
      "role": "assistant", "content": null,
      "tool_calls": [{
        "id": "call_abc123",                    // ← 回填时要对应
        "function": { "name": "get_weather",
                      "arguments": "{\"city\": \"北京\"}" }
      }]
    }
  }]
}
```

**阶段三：你执行，结果回填**——结果作为 `role: "tool"` 消息、携带 `tool_call_id` 追加回消息数组，再次请求：

```python
messages.append({"role": "tool", "tool_call_id": "call_abc123", "content": "{\"temp\": 25}"})
resp = client.chat.completions.create(model=..., messages=messages, tools=tools)
```

模型看到结果后可继续思考、继续调用，直到 `finish_reason: stop`。

> **练习 1.2**：给模型 `add` 和 `multiply` 两个工具，问"计算 (2+3)*4"。观察需要几轮——这就是多步工具调用，是所有 Agent 的雏形。

### 3.3 工程要点（踩坑总结）

1. **arguments 是字符串**：必须 `json.loads` 解析，且要处理解析失败（模型可能给非法 JSON）；
2. **一次响应可能多个 tool_calls**：循环要支持并行执行——**只读调用可并发，写操作串行**（生产框架的调度契约，DSH 的实现见第 3 篇 5.5）；
3. **失败处理的正道**：把错误作为工具结果回填（`result = f"ERROR: {e}"`），**让模型看到错误自己纠正**——这是 Agent 框架的标准做法，比代码里硬重试更有效；
4. **工具数量控制**：一个 Agent 可见工具建议 ≤ 20 个，多了模型"选择困难"甚至忽略工具；
5. **描述第一句要点题**：`"查询指定城市的当前天气，当用户问天气时使用"` 优于 `"天气工具"`。

## 4. 经典架构模式：如何组织多轮调用

函数调用是"原子操作"，架构模式是"组合方式"。四种模式按复杂度递进：

### 4.1 ReAct（推理 + 行动交替）——最经典

让模型在上下文里"自言自语"（Thought），再调工具（Action），看到结果（Observation）继续。**推理文本保留在上下文里**是灵魂——既帮模型保持连贯，也让人类能审计它在想什么：

```text
Thought: 我需要先知道两个城市的纬度再比较。
Action: geocode(city="Paris")
Observation: 48.8566° N
Thought: 巴黎 48.86°N，查伦敦。
Action: geocode(city="London")
Observation: 51.5074° N
Final: 伦敦（51.5°N）比巴黎（48.9°N）高。
```

### 4.2 Plan-and-Execute（先规划再执行）——控制复杂任务

ReAct 走一步看一步，复杂任务易迷失。改进：先让模型**一次性产出完整计划**，再逐步执行、按需修订。DSH 的 **plan mode** 就是它的工程化：先只读探索 → 产出计划 → **人工审批** → 才允许改文件（第 3 篇详述）。

### 4.3 Reflexion（反思重试）——提高成功率

失败后让模型**分析失败原因、写进上下文、再重试**：

```python
for attempt in range(3):
    result = run_agent(task)
    if success(result): return result
    reflection = ask_model(f"上次失败：{result.error}，分析原因并给出修正方案")
    history.append(f"[反思] {reflection}")
```

### 4.4 2026 年生产 Agent 循环的标准形态

```text
规划(Plan，复杂任务先出计划，可修订)
  → 执行循环(Thought → Action → Observation；失败插 Reflection；上下文过大触发 Compaction)
  → 交付(结构化输出 + 必要时人工确认)
```

DSH 的实现就是这张图的工程化：计划模式、agent loop、压缩（Compaction）、目标（长任务）、审批。**先把这张图看懂，后面每一篇都在往图里填实现细节。**

## 5. 记忆与上下文管理：Agent 最大的工程挑战

### 5.1 问题本质：上下文是"物理定律"

回到 1.3 的认知：上下文窗口有限是注意力的物理限制。Agent 每轮都会把工具结果塞回上下文，**对话越长，可用窗口越少，token 成本越高**。这不是优化问题，是"内存有限"问题——所以叫"记忆管理"。三种主流策略各有其原理与代价：

### 5.2 策略一：截断（Truncation）——最便宜，最危险

原理：直接丢旧消息。代价：**早期关键信息可能被丢掉**（任务定义、用户偏好）。工程改进是"分层保留"——系统提示词和任务定义永远在，只剪对话历史：

```python
KEEP = 4   # 开头保留条数
messages = messages[:KEEP] + messages[-16:]
```

### 5.3 策略二：压缩（Compaction）——主流方案

原理：用模型把旧对话**总结成摘要**替换原文——"记忆"变成了"笔记"。代价：摘要会丢细节（精确数字、工具输出），所以生产框架只压缩"不重要的旧历史"，工具结果单独走"修剪"（砍头留尾，DSH 参数：超过 8192 字符才修剪，保留头 4096 + 尾 1024 字符——头部通常含结构，尾部通常含结论）。

### 5.4 策略三：检索（RAG）——知识问答的标配

原理：把资料切块 → 向量化 → 查询时只把**最相关的几块**放进上下文。这是"记忆"的第三种形态：**不记全部，记索引**。最小实现（15 行）：

```python
import numpy as np
def embed(texts):  # 调 embedding API
    r = client.embeddings.create(model="text-embedding-3-small", input=texts)
    return [d.embedding for d in r.data]

chunks = split_markdown("docs.md")          # 离线切块
vecs = np.array(embed(chunks))              # 每块一个向量

def retrieve(query, k=3):                   # 在线检索 top-k
    qv = np.array(embed([query])[0])
    return [chunks[i] for i in np.argsort(vecs @ qv)[-k:][::-1]]
```

RAG 的三个坑（2026 年共识）：切块粒度（500~1000 字符较稳）、检索质量（先召回再精排）、**来源标注**（回答必须可溯源，合规必需）。

### 5.5 记忆的类型学（进阶视野）

| 记忆类型 | 载体 | 例子 | 工程实现 |
|---|---|---|---|
| 工作记忆 | 上下文窗口 | 当前任务中间结果 | 对话历史（不落地） |
| 长期记忆 | 外部存储 | 用户偏好、项目背景 | 向量库/数据库 + 检索 |
| 程序性记忆 | 代码/配置 | "这类 bug 怎么修" | 工具、Skills、预设（DSH 的 skills 机制） |

## 6. 一个最小 Agent：只保留核心循环（35 行）

把 §3 的知识浓缩成最小可运行 Agent。**刻意省略**了工具实现的细节（用两个简单函数代替），让你把注意力放在唯一的重点——**循环本身**：

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

# ── Agent 循环：这才是全部"智能"发生的地方 ──
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
1. **信号灯**：`finish_reason` / `tool_calls is None` 决定循环是否继续；
2. **契约**：工具只暴露 name/description/parameters 给模型，实现是黑盒；
3. **回填闭环**：assistant 消息（含 tool_calls）和 tool 消息（含 call_id）必须都进历史；
4. **截断**：`str(result)[:3000]`——上下文保护从第一行代码就开始。

> **练习 1.3**：跑通后回答三个问题：① 为什么 `content` 要截断？② 如果工具抛异常会发生什么？③ 把任务换成需要 5 步以上的，观察循环如何协作。（答案提示：① 上下文有限且计费；② 循环崩溃——生产做法是捕获后把错误回填给模型；③ 每步一次往返，这就是 PTC 模式要解决的问题。）

## 7. 从最小 Agent 到生产级框架：差距清单

35 行能工作，但离 DSH 这样的生产框架差着一个工程时代。**这张表就是整个 Agent 领域的"待办清单"**，也是本系列后续每一篇的主题：

| 问题 | 最小 Agent | 生产级框架（DSH） |
|---|---|---|
| 工具安全 | 裸 `eval`，什么都能干 | 沙箱、文件策略、命令白名单、人工审批 |
| 上下文 | 硬截断 3000 字符 | 压缩 + 修剪 + token 计量 + KV 缓存友好组织 |
| 多步效率 | 每步一次往返 | **PTC 模式**：一次程序执行跑完多步 |
| 错误恢复 | 靠模型自己 | 结构化错误（ToolCallError）、失败注入上下文 |
| 可观测 | print 调试 | 事件流、会话回放、UI 渲染 |
| 扩展性 | 改代码加工具 | **插件系统**：能力按行声明、热插拔（Cordis） |
| 并行 | 无 | 只读并发 / 写串行调度契约 |
| 多 Agent | 无 | 子代理、工作流编排 |
| 记忆 | 无持久化 | 目标（goal）、计划模式、Skills |

## 检验：读完本节，你能回答吗

1. **为什么说“函数调用本质是文本生成”？** 这解释了哪个工程现象（幻觉参数）？
2. 上下文窗口为什么存在（物理机制）？KV 缓存对提示词工程的三条铁律是什么？
3. 工具调用失败时，为什么“把错误回填给模型”比代码里硬重试更有效？
4. 截断 / 压缩 / 检索三种记忆策略各有什么代价？分别在什么场景用？
5. 35 行 Agent 循环里，“信号灯”是什么？assistant 消息和 tool 消息为什么都必须进历史？

（答案都在上文对应小节：1→§3.1，2→§1.3，3→§3.3，4→§5，5→§6）

## 本章小结

- LLM 是**按概率采样**的文本生成器：token 化、温度采样、注意力决定了上下文上限，KV 缓存决定了"前缀稳定"的工程铁律；
- **函数调用本质是文本生成**：schema 描述 → `tool_calls` 请求 → 你执行 → 回填；失败的正道是"把错误给模型看"；
- **Agent = LLM + 工具 + 循环**：ReAct（交替）、Plan-and-Execute（先规划）、Reflexion（反思重试）三种模式组装成生产循环；
- **记忆三板斧**：截断（便宜但危险）、压缩（摘要替换）、检索（只记索引）——各有原理与代价；
- 35 行核心循环是学习基线；生产框架解决安全、效率、可观测、可扩展——**这就是 DSH 存在的意义**。

**下一篇**：[第 2 篇 · Agent 框架横评与 DSH 的定位](02-框架横评与DSH定位.md)