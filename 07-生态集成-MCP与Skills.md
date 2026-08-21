# 第 7 篇 · 生态集成：MCP 与 Skills

> **搭建路线**：第 7 块积木——扩展"手"与"做事方法"：MCP 让外部系统以通用方式接入，Skills 让任务 SOP 以渐进披露方式进入上下文。顺便看清 2026 框架版图。
> **本篇目标**：分清 MCP 与 Skills 的分工；掌握协议生命周期、管理器工程要点、渐进披露；在框架版图中定位自己的系统。
> **难度**：⭐⭐⭐

---

## 1. 两个问题，两个答案

你的 Agent 已经能循环、能用工具、能管上下文、能记、能规划。但继续往前走会出现两个问题（整合自 Tritium《Build An Agent From Scratch》[5.3]，CC BY-NC-SA）：

**问题一：工具生态不该由项目自己无限维护。** 今天接 GitHub、明天接 Linear、后天接公司内部文档系统——每接一个就在仓库里写 schema、鉴权、调用逻辑、错误格式化、测试。Agent runtime 会变成越来越臃肿的集成仓库。**这不是 Agent Loop 应该承担的复杂度。**

**问题二："怎么做一类任务"的经验不该写死进默认 prompt。** 做 code review 时应该先读 diff 再按风险排序输出；写文章时应该先读前文风格再抽取代码变更；做迁移时应该先建立兼容性清单再逐步替换——这些是某类任务的**操作手册（SOP）**。全部塞进 system prompt，模型每轮都要背着一堆当前用不到的流程说明。

**MCP 与 Skills 正好互补**：MCP 扩展 Agent 的"手"（外部系统接口），Skills 扩展 Agent 的"做事方法"（任务 SOP）。一个解决"接口从哪来"，一个解决"方法怎么教"。

## 2. MCP：协议、生命周期、传输

### 2.1 为什么需要协议

没有 MCP 之前，每个 Agent 都要为每个工具写专属接入代码——生态碎片化。MCP 统一接入方式：**任何 Agent 可接入任何 MCP Server**。为什么选 JSON-RPC 2.0：简单通用，请求-响应模型与工具调用天然契合。

### 2.2 会话生命周期

```text
1. initialize 握手       客户端声明能力，Server 声明协议版本与能力
2. notifications/initialized  Server 确认
3. tools/list            客户端发现工具（可反复调用，支持增量更新）
4. tools/call            调用工具（每次一个 JSON-RPC 请求）
5. 资源同步              resources/list → subscribe → notifications（变更通知）
6. 关闭                  客户端退出（stdio 管道关闭 / HTTP 连接断开）
```

### 2.3 三种传输的取舍

| 传输 | 适合 | 特点 |
|---|---|---|
| stdio | 本地子进程 | 最简单，进程级隔离；Agent 拉起 Server 子进程 |
| HTTP + SSE | 远程服务 | 传统流式，连接可复用 |
| Streamable HTTP | 远程服务（2025+ 主流） | 单一端点双向流，云部署友好 |

### 2.4 最小 Server（10 行，完整可运行）

```ts
import { McpServer } from '@modelcontextprotocol/sdk/server/mcp.js'
import { StdioServerTransport } from '@modelcontextprotocol/sdk/server/stdio.js'

const server = new McpServer({ name: 'issue-tracker', version: '1.0.0' })

// 工具：查询内部工单系统
server.tool('get_issue', { id: { type: 'string', description: '工单 ID' } },
  async ({ id }) => ({
    content: [{ type: 'text', text: JSON.stringify(await internalApi.fetchIssue(id)) }],
  }))

// 资源：暴露只读配置
server.resource('config://deploy', async (uri) => ({
  contents: [{ uri, text: JSON.stringify(getConfig()) }],
}))

await server.connect(new StdioServerTransport())   // stdio 传输：Agent 以子进程拉起
```

### 2.5 管理器工程要点（接入 Agent runtime 时）

- **管理器**启动 stdio server，发现工具并映射为内部 AgentTool（统一 schema 与执行接口）；
- **工具过滤**：enabledTools / disabledTools 控制暴露范围——**不是发现什么就暴露什么**；
- **诊断不炸循环**：server 启动失败、工具调用失败都转换成可读错误进上下文，而不是让 Agent 崩溃（第 2 篇"错误进循环"的扩展）；
- **权限联动**：MCP 工具同样进入 read / write / execute 分级治理——外部工具不豁免安全策略。

### 2.6 安全边界

MCP Server 的能力 = 你授予 Agent 的能力。**接一个恶意的 MCP Server = 给 Agent 一个任意执行的后门**。审查清单：只接可信来源；Server 用最小权限账号；敏感数据在 Server 侧脱敏；所有调用留审计。

## 3. Skills：渐进披露的任务 SOP

### 3.1 形态

Skill 是一个带 frontmatter 的 Markdown 文件（SKILL.md），描述某类任务的详细流程：

```text
---
name: repo-review
description: Review a repository change for bugs and missing tests.
---

Read the diff, prioritize concrete risks, and report findings first.
Then verify fixes with tests before concluding.
```

### 3.2 三个关键设计

1. **description 必填**：模型一开始只看到技能索引（名称+描述），靠描述判断是否匹配当前任务——没有描述就没有可发现入口；
2. **渐进披露**：系统提示词只注入技能索引（轻量），模型判断需要时再**主动加载全文**——避免每轮背着一堆用不到的 SOP；
3. **隐藏隐式调用**：某些 skill 不希望模型自动调用（如安全审计流程），可用 frontmatter 的 disable-model-invocation 标记。

### 3.3 与记忆的关系

Skills 就是第 5 篇的"程序性记忆"：把"怎么做一类任务"的经验做成**可发现的资源**（而不是塞进提示词或记忆文件）。区别：记忆是"事实"，Skills 是"方法"。

## 4. 框架版图：你现在站在哪里

到本篇为止，你已经亲手搭过一个 Agent 的完整骨架。站在这个高度看 2026 框架版图：

| 层 | 代表 | 与你的搭建路线的对应 |
|---|---|---|
| 编排框架（库） | LangGraph、OpenAI Agents SDK | 你的循环 + Planner 是同一思想的代码形态 |
| 成品 Agent（运行时） | Claude Code、**DSH** | 你的 9 块积木的生产级拼装（下篇） |
| 协议/标准 | **MCP** | 本篇第 2 节 |
| 基础设施 | 评测、可观测、向量库 | 第 9 篇企业实战 |

两个代表性框架的核心思想（理解它们=理解框架层）：

```python
# LangGraph：图状态机——"把流程画出来"
g = StateGraph(AgentState)
g.add_node("agent", call_model)                # 节点：调 LLM
g.add_node("tools", call_tools)                # 节点：执行工具
g.add_conditional_edges("agent", route, {"tools": "tools", "end": END})
g.add_edge("tools", "agent")                   # 工具结果回到 agent——循环的"边"
# 哲学：复杂流程（分支/循环/人工介入/并行）在图里一目了然，且天然可持久化
```

```python
# OpenAI Agents SDK：Agent 是可组合单元 + Handoffs（任务转交）
agent = Agent(name="assistant", instructions="你是助手。",
              tools=[add_tool, multiply_tool])
result = Runner.run_sync(agent, "计算 (2+3)*4")   # 循环被封装在 Runner 里
triage = Agent(name="triage", handoffs=[coder_agent, reviewer_agent])
# 哲学：Agent 就是函数，组合/转交/编排都由框架处理
```

> **选型建议**：你已经会搭骨架，选框架就是选"哪块积木买现成的"——LangGraph 买"图编排"、MCP 买"工具生态"、DSH 买"全套生产实现"。**先会搭，再会选**。

## 5. DSH 参照：dsh-mcp-client 与 skills

| 本篇概念 | DSH 的对应 |
|---|---|
| MCP 管理器 | `dsh-mcp-client` 插件：组合里声明 Server，工具自动进入 Agent 目录 |
| Skills | `dsh-skill-filesystem` + `dsh-tool-skill`：SKILL.md 按需加载，避免提示词膨胀 |
| 权限联动 | MCP 工具进入 Agent 目录后同样受沙箱/审批约束 |

```yaml
# DSH 组合中的 MCP 接入（示意）
- id: mcp-issues
  name: '@deepseek-ai/dsh-mcp-client'
  config:
    servers:
      issues:
        command: "node"
        args: ["mcp-server/dist/index.js"]
        transport: stdio
```

## 6. 练习与常见坑

> **练习 7.1**：用 §2.4 的模板写一个 MCP Server（任意包装一个本地 API），再用命令行客户端（或 DSH 的 dsh-mcp-client）连上它，跑通 tools/list 与 tools/call。
> **练习 7.2**：写一个 repo-review 的 SKILL.md，给第 2 篇的循环加"技能索引 → 按需加载全文"机制，观察模型在匹配任务时是否主动加载。

**常见坑**：
1. **发现即暴露**：MCP Server 发现 50 个工具全部注册——模型"选择困难"（第 3 篇工具数量铁律）；
2. **MCP 工具豁免安全策略**：外部工具照样能跑危险操作——必须进 read/write/execute 治理；
3. **Skill 全文注入提示词**：渐进披露被绕过，提示词又膨胀了；
4. **description 敷衍**：模型无法判断何时该用这个 skill——等于没有。

## 7. 检验：读完本篇，你能回答吗

1. MCP 与 Skills 的分工是什么？（手 vs 做事方法）
2. MCP 会话生命周期六阶段是什么？三种传输怎么选？
3. 渐进披露的三步是什么？description 为什么必填？
4. MCP 管理器四个工程要点是什么？"诊断不炸循环"对应哪篇的哪个原则？
5. 为什么"接恶意 MCP Server = 给 Agent 一个后门"？
6. LangGraph 与 OpenAI Agents SDK 的核心哲学各是什么？
7. 你搭过的 9 块积木，在 LangGraph / DSH 里各是"买的现成"的哪块？

（答案都在上文：1→§1，2→§2，3→§3，4→§2.5，5→§2.6，6→§4，7→§4）

## 本章小结

第 7 篇拼上了生态扩展：手可以无限接（MCP），方法可以按需教（Skills）。至此，你已经从理论到实践搭出了一个 Agent 的完整骨架。下一篇，看看生产级实现——DSH——是怎么把这些积木拼成产品的。

**下一篇**：[第 8 篇 · DSH 生产级参照](08-DSH生产级参照.md) —— 9 块积木的生产级拼法：Cordis、双平面、预设、PTC。
