# 第 7 篇 · 生态集成：MCP 与 Skills

> **搭建路线**：第 7 块积木——扩展"手"与"做事方法"：MCP 让外部系统以通用方式接入，Skills 让任务 SOP 以渐进披露方式进入上下文。顺便看清 2026 框架版图。
> **本篇目标**：分清 MCP 与 Skills 的分工；掌握渐进披露与 MCP 管理器工程要点。
> **难度**：⭐⭐⭐

---

## 1. 两个问题，两个答案

你的 Agent 已经能循环、能用工具、能管上下文、能记、能规划。但继续往前走会出现两个问题（整合自 Tritium《Build An Agent From Scratch》[5.3]，CC BY-NC-SA）：

1. **工具生态不该由项目自己无限维护**：今天接 GitHub、明天接 Linear、后天接内部文档系统——每接一个就在仓库里写 schema、鉴权、调用逻辑，Agent runtime 会变成越来越臃肿的集成仓库；
2. **"怎么做一类任务"的经验不该写死进默认 prompt**：code review 的流程、写文章的流程、做迁移的流程——全部塞进 system prompt 会让模型每轮背着一堆当前用不到的说明。

**MCP 与 Skills 正好互补**：MCP 扩展 Agent 的"手"（外部系统接口），Skills 扩展 Agent 的"做事方法"（任务 SOP）。

## 2. MCP：协议、传输与安全

### 2.1 协议形态

MCP（Model Context Protocol）统一 Agent 接入工具/数据的方式：JSON-RPC 2.0 + 三种传输（stdio 本地子进程 / HTTP+SSE / Streamable HTTP），三种原语——**Tools**（能做）、**Resources**（能读）、**Prompts**（能教）。

### 2.2 一个最小 Server（10 行）

```ts
import { McpServer } from '@modelcontextprotocol/sdk/server/mcp.js'
import { StdioServerTransport } from '@modelcontextprotocol/sdk/server/stdio.js'

const server = new McpServer({ name: 'issue-tracker', version: '1.0.0' })
server.tool('get_issue', { id: { type: 'string' } }, async ({ id }) => ({
  content: [{ type: 'text', text: JSON.stringify(await internalApi.fetchIssue(id)) }],
}))
await server.connect(new StdioServerTransport())
```

### 2.3 管理器工程要点（接入 Agent runtime 时）

- **管理器**启动 stdio server，发现工具并映射为内部 AgentTool（统一 schema 与执行接口）；
- **工具过滤**：enabledTools / disabledTools 控制暴露范围——不是发现什么就暴露什么；
- **诊断不炸循环**：server 启动失败、工具调用失败都转换成可读错误进上下文，而不是让 Agent 崩溃（第 2 篇"错误进循环"的扩展）；
- **权限联动**：MCP 工具同样进入 read / write / execute 分级治理——外部工具不豁免安全策略。

### 2.4 安全边界

MCP Server 的能力 = 你授予 Agent 的能力。**接一个恶意的 MCP Server = 给 Agent 一个任意执行的后门**。审查清单：只接可信来源；Server 用最小权限账号；敏感数据在 Server 侧脱敏；所有调用留审计。

## 3. Skills：渐进披露的任务 SOP

Skill 是一个带 frontmatter 的 Markdown 文件（SKILL.md），描述某类任务的详细流程：

```text
---
name: repo-review
description: Review a repository change for bugs and missing tests.
---

Read the diff, prioritize concrete risks, and report findings first.
```

三个关键设计：

1. **description 必填**：模型一开始只看到技能索引（名称+描述），靠描述判断是否匹配当前任务——没有描述就没有可发现入口；
2. **渐进披露**：系统提示词只注入技能索引（轻量），模型判断需要时再**主动加载全文**——避免每轮背着一堆用不到的 SOP；
3. **隐藏隐式调用**：某些 skill 不希望模型自动调用（如安全审计），可用 frontmatter 标记 disable-model-invocation。

## 4. 框架版图：你现在站在哪里

到本篇为止，你已经亲手搭过一个 Agent 的完整骨架。站在这个高度看 2026 框架版图（对照第 1 篇的模块全景）：

| 层 | 代表 | 与你的搭建路线的对应 |
|---|---|---|
| 编排框架（库） | LangGraph（图状态机）、OpenAI Agents SDK（Handoff） | 你的循环 + Planner 是同一思想的代码形态 |
| 成品 Agent（运行时） | Claude Code、**DSH** | 你的 9 块积木的生产级拼装（下篇） |
| 协议/标准 | **MCP** | 本篇第 2 节 |
| 基础设施 | 评测、可观测、向量库 | 第 9 篇企业实战 |

> 选型建议：你已经会搭骨架，选框架就是选"哪块积木买现成的"——LangGraph 买"图编排"、MCP 买"工具生态"、DSH 买"全套生产实现"。**先会搭，再会选**。

## 5. DSH 参照：dsh-mcp-client 与 skills

| 本篇概念 | DSH 的对应 |
|---|---|
| MCP 管理器 | `dsh-mcp-client` 插件：组合里声明 Server，工具自动进入 Agent 目录 |
| Skills | `dsh-skill-filesystem` + `dsh-tool-skill`：SKILL.md 按需加载，避免提示词膨胀 |
| 权限联动 | MCP 工具进入 Agent 目录后同样受沙箱/审批约束 |

## 6. 检验：读完本篇，你能回答吗

1. MCP 与 Skills 的分工是什么？（手 vs 做事方法）
2. 渐进披露的三步是什么？description 为什么必填？
3. MCP 管理器四个工程要点是什么？"诊断不炸循环"对应哪篇的哪个原则？
4. 为什么"接恶意 MCP Server = 给 Agent 一个后门"？
5. 你搭过的 9 块积木，在 LangGraph / DSH 里各是"买的现成"的哪块？

## 本章小结

第 7 篇拼上了生态扩展：手可以无限接（MCP），方法可以按需教（Skills）。至此，你已经从理论到实践搭出了一个 Agent 的完整骨架。下一篇，看看生产级实现——DSH——是怎么把这些积木拼成产品的。

**下一篇**：[第 8 篇 · DSH 生产级参照](08-DSH生产级参照.md) —— 9 块积木的生产级拼法：Cordis、双平面、预设、PTC。
