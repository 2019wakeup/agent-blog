# 第 2 篇 · Agent 框架横评与 DSH 的定位

> **难度**：⭐⭐
> **本篇目标**：看清 2026 年 Agent 工具版图——哪些是"框架库"、哪些是"产品"、哪些是"协议"。每个框架只给**最能说明设计思想**的核心片段，重点讲"为什么这样设计"。

---

## 1. 四层版图：先分清你在哪一层

| 层 | 代表 | 你写什么 | 类比 |
|---|---|---|---|
| ① 编排框架（库） | LangGraph、OpenAI Agents SDK、Google ADK、AutoGen、CrewAI | 用代码定义循环/状态/协作 | Web 框架（Express） |
| ② 成品 Agent（产品/运行时） | Claude Code、Codex、Cursor、**DSH** | 直接用，或改装 | 成品应用（IDE） |
| ③ 协议/标准 | **MCP** | 写 Server 接入能力 | USB |
| ④ 基础设施 | 向量库、评测、可观测 | 配置/脚本 | 数据库/监控 |

> 常见误区："LangGraph vs Claude Code"不是同一层——LangGraph 是**你用来写 Agent 的库**，Claude Code 是**别人写好的 Agent**。DSH 属于②，但它本身也是①（你能改装它）。**选型第一步：分清你在哪一层**。

## 2. 编排框架：两个代表，两种设计哲学

### 2.1 LangGraph：图状态机——"把流程画出来"

**设计哲学**：Agent 流程本质是"状态在节点间流动的有向图"。节点 = 函数（调 LLM/执行工具/改状态），边 = 条件路由。**为什么用图**：复杂流程（分支、循环、人工介入、并行）在图里一目了然，且天然可持久化（保存状态 = 断点续跑）。

核心只有 5 行（其余是样板）：

```python
g = StateGraph(AgentState)                    # 状态在节点间流动
g.add_node("agent", call_model)               # 节点：调 LLM（可能返回工具调用）
g.add_node("tools", call_tools)               # 节点：执行工具
g.add_conditional_edges("agent", route, {"tools": "tools", "end": END})
g.add_edge("tools", "agent")                  # 工具结果回到 agent —— 循环的"边"
```

**为什么选它**：控制粒度最细、生态最大。**为什么不用它**：简单任务画图是负担（样板多、概念多：State/Node/Edge/Checkpointer）。

### 2.2 OpenAI Agents SDK：轻量 + 转交——"把 Agent 当函数"

**设计哲学**：Agent 就是一个可组合的单元（模型+指令+工具），核心创新是 **Handoffs（转交）**——任务可以从一个 Agent 转交给另一个（按技能分工）。

```python
agent = Agent(name="assistant", instructions="你是助手。",
              tools=[add_tool, multiply_tool])   # 工具就是普通函数
result = Runner.run_sync(agent, "计算 (2+3)*4")   # 循环被封装在 Runner 里

triage = Agent(name="triage", handoffs=[coder_agent, reviewer_agent])  # 转交
```

**为什么选它**：API 极简、上手最快。**为什么不用它**：绑定 OpenAI 生态；复杂状态机能力弱于 LangGraph。

### 2.3 其余三家一句话

| 框架 | 一句话定位 | 适合 |
|---|---|---|
| Google ADK | 代码优先多 Agent，与 Vertex AI 深度集成 | 上云 GCP 的团队 |
| AutoGen（微软） | 多 Agent"对话式"协作（A 的输出喂给 B） | 研究多智能体协作 |
| CrewAI | 角色化团队（分析师/写手/审查员） | 原型验证 |

> **选型建议**：主学一个、学透一个。框架的底层（循环、状态、工具）是相通的——第 1 篇的 35 行 Agent 是所有框架的"母版"。

## 3. 成品编码 Agent 对比：黑盒 vs 透明

| | Claude Code | Codex | Cursor | DSH |
|---|---|---|---|---|
| 开源 | ❌ | CLI 开源（服务闭源） | ❌ | ✅ 完全开源 |
| 架构可见性 | 黑盒 | 半开 | 黑盒 | **一切皆插件，可读可改** |
| 扩展 | 受限插件 | 受限插件 | 规则配置 | **任意预设/工具/提示词** |
| 卖点 | 工程完成度高 | 模型强 | 编辑器集成 | **透明、可自改、省 token** |

**为什么"黑盒"是学习者的最大障碍**：Agent 领域 80% 的工程知识在**运行时内部**（提示词组织、工具调度、上下文策略）——闭源产品你永远看不到。DSH 的价值就是把这一切开源。

## 4. MCP：为什么需要"协议"

### 4.1 问题与解法

**问题**：每个 Agent 都要为每个工具写专属接入代码——生态碎片化。**解法**：统一协议（MCP），任何 Agent 可接入任何 MCP Server（暴露 Tools/Resources/Prompts 的独立进程）。**为什么是 JSON-RPC 2.0 + 三种传输**：JSON-RPC 简单通用（请求-响应模型与工具调用天然契合）；stdio 适合本地子进程，Streamable HTTP 适合远程服务。

### 4.2 一个最小 Server（10 行）

```ts
// npm i @modelcontextprotocol/sdk
import { McpServer } from '@modelcontextprotocol/sdk/server/mcp.js'
import { StdioServerTransport } from '@modelcontextprotocol/sdk/server/stdio.js'

const server = new McpServer({ name: 'issue-tracker', version: '1.0.0' })
server.tool('get_issue', { id: { type: 'string' } }, async ({ id }) => ({
  content: [{ type: 'text', text: JSON.stringify(await internalApi.fetchIssue(id)) }],
}))
await server.connect(new StdioServerTransport())
```

### 4.3 安全边界（企业必读）

MCP Server 的能力 = 你授予 Agent 的能力。**接一个恶意的 MCP Server = 给 Agent 一个任意执行的后门**。审查清单：只接可信来源；Server 用最小权限账号；敏感数据在 Server 侧脱敏；所有调用留审计。

> DSH 自带 `dsh-mcp-client`：组合里声明 Server，工具自动进入 Agent 目录——"写一次 Server，处处复用"（评审 Agent、客服 Agent、运维 Agent 共享同一批 MCP Server）。

### 4.4 从协议到工程：MCP 管理器与 Skills

> 本节整合自 Tritium《Build An Agent From Scratch》[5.3] 篇（CC BY-NC-SA，https://www.tritium.work）。

协议之外还有两个工程问题：**工具生态不该由项目自己无限维护**（每接一个系统就在仓库里写 schema/鉴权/调用逻辑，Agent runtime 会越来越臃肿）；**“怎么做一类任务”的经验也不该全部写死进 system prompt**（每类任务的 SOP 会让 prompt 越来越长）。

MCP 与 Skills 正好互补：**MCP 扩展 Agent 的“手”（外部系统接口），Skills 扩展 Agent 的“做事方法”（任务 SOP）**。

#### Skills：渐进披露的任务 SOP

Skill 是一个带 frontmatter 的 Markdown 文件（SKILL.md），描述某类任务的详细流程：

```text
---
name: repo-review
description: Review a repository change for bugs and missing tests.
---

Read the diff, prioritize concrete risks, and report findings first.
```

三个关键设计：

1. **description 是必填的**——模型一开始只看到技能索引（名称+描述），靠描述判断是否匹配当前任务，没有描述就没有可发现入口；
2. **渐进披露**：系统提示词里只注入技能索引（轻量），模型判断需要时再**主动加载全文**——避免让模型每轮背着一堆当前用不到的 SOP；
3. **隐藏隐式调用**：某些 skill 不希望模型自动调用（如安全审计流程），可用 frontmatter 的 disable-model-invocation 标记。

> DSH 的 skills 机制（第 3 篇提及）就是同一思想：SKILL.md 按需加载，避免提示词膨胀。

#### MCP 管理器：外部工具进入运行时

接入 MCP 的工程要点（对应 DSH 的 dsh-mcp-client）：

- **管理器**启动 stdio MCP server，发现其工具并映射为内部 AgentTool（统一 schema 与执行接口）；
- **工具过滤**：enabledTools / disabledTools 控制暴露范围——不是发现什么就暴露什么；
- **诊断不炸循环**：server 启动失败、工具调用失败都转换成可读错误进上下文，而不是让 Agent 崩溃；
- **权限联动**：MCP 工具同样进入 read / write / execute 分级治理——外部工具不豁免安全策略。

> “与时代接轨不是追热点”：MCP 和 Skills 之所以重要，是因为它们正好落在 Harness 的边界上——一个扩展“可发现的能力”，一个扩展“可复用的方法”，都让 Agent 系统在不变的循环结构上获得新的能力来源。

## 5. 基础设施（入门就要有意识）

| 类 | 代表 | 一句话 |
|---|---|---|
| 评测 | SWE-bench（基准）、自建评测集 + CI | 没有评测 = 没有测试 |
| 可观测 | Langfuse、OpenTelemetry GenAI 约定 | 每次思考/调用/token 都可查 |
| 向量库 | pgvector（入门）、Qdrant、Milvus | RAG 的地基 |

## 6. DSH 的定位：为什么"框架 + 产品"两层它都沾

- **作为成品（②）**：`dsh web` 起 Web UI，`dsh --profile headless "任务"` 跑无头任务；
- **作为框架（①）**：通过插件组装 Agent——能力以 `cordis.yml` 插件行为单位（第 3 篇详述）；
- **作为平台**：Fiber 生命周期托管让动态插件可安全热插拔。

> **一句话定位**：DSH 是把"Agent 运行时"本身做成乐高的开源项目——每块能力（工具、提示词、沙箱、记忆、审批）都是一块积木，接口是 Cordis 插件系统。**适合"拿来就用"，也适合"拆开研究"**。

## 7. 选型决策树

```text
你的需求是什么？
├─ 研究/教学用小 Agent        → OpenAI Agents SDK 或手写（第 1 篇 35 行）
├─ 复杂业务流程编排           → LangGraph / Google ADK
├─ 团队多人用的编码 Agent     → 优先开源可定制：DSH
├─ 给 Agent 接第三方能力      → 学 MCP，找现成 Server
└─ 企业级交付（安全/评测/CI） → 第 4 篇实战路线：以 DSH 为平台
```

## 检验：读完本节，你能回答吗

1. Agent 工具的四层版图是什么？为什么“LangGraph vs Claude Code”是伪对比？
2. LangGraph 的“图哲学”解决了什么问题？它的代价是什么？
3. OpenAI Agents SDK 的核心创新（Handoffs）解决什么问题？
4. MCP 解决什么行业问题？为什么选 JSON-RPC 作为协议基础？
5. DSH 与 Claude Code 的本质区别是什么？为什么说 DSH 横跨“成品 + 框架”两层？

## 本章小结

- Agent 工具分四层：编排框架 / 成品 Agent / 协议（MCP）/ 基础设施——**选型先分清你在哪一层**；
- LangGraph（图哲学）与 OpenAI Agents SDK（组合+转交哲学）是框架层两大代表，各给 5-6 行核心；
- MCP 用统一协议解决生态碎片化，"协议"是 Agent 时代的 USB；
- DSH 横跨"成品 + 框架"：开源、透明、可自改，是理解 Agent 内部构造的最佳教材。

**下一篇**：[第 3 篇 · DSH 深度解剖](03-DSH深度解剖.md)