# 第 4 篇 · 企业级实战：AutoResearcher 科研 Agent

> **难度**：⭐⭐⭐⭐（本系列核心，直达终极目标）
> **本篇目标**：完整走通一个**企业级科研 Agent 项目**——文献调研、实验运行、数据分析，**前后端分离**、后端部署在 Ubuntu 远程主机。重点不是代码本身（工程代码都在 `autoresearcher/` 项目里），而是**每个企业级决策背后的原理与权衡**。代码只保留最能说明决策的关键片段。

---

## 4.0 项目总览：三层架构（前后端分离的骨架）

```text
┌─────────────────────────────────────────────┐
│ 前端层（任何机器可访问）                      │
│  Web 面板：任务创建/结果/日志流 —— 只调后端 API │
└───────────────┬─────────────────────────────┘
                │ REST + WebSocket（Bearer token）
┌───────────────▼─────────────────────────────┐
│ 后端层（Ubuntu 主机）                        │
│  API 服务（FastAPI）                        │
│   ├─ 任务注册表（持久化）/鉴权/日志流         │
│   └─ spawn dsh --profile <P> "任务"          │
└───────────────┬─────────────────────────────┘
                │
┌───────────────▼─────────────────────────────┐
│ Agent 核心（DSH headless）                   │
│  AutoResearcher 预设 + 科研工具插件           │
│  沙箱/审批/预算账本/审计                      │
└─────────────────────────────────────────────┘
```

**外卖平台类比**：前端面板是**点餐 App**，后端 API 是**平台服务器**，Agent 核心是**商家厨房**。App 永远不直接进厨房（前端不碰 Agent）——所有下单、状态、出餐都经过平台；厨房可以换（换预设/换模型），App 不用改。**分离的本质是“谁跟谁说话”的约定（契约表），而不是代码在哪个目录**。

**三条设计原则**（理解它们比记住架构图更重要）：
1. **前端不碰 Agent**——所有交互走后端 API。前端可以随便换（React/Vue/CLI），后端和 Agent 核心完全不动；
2. **Agent 核心是子进程**——后端用 `spawn dsh --profile headless "任务"` 隔离运行：任务崩溃不影响服务，日志落盘可审计；
3. **远程部署 = 进程托管**——Docker/systemd 管后端生命周期，Nginx 管静态托管与反代（4.8）。

### 一次科研任务的完整时序（先看这张图，再看 4.1~4.9 的实现）

```text
前端面板            后端 API              Agent 核心（DSH）          数据/审计
   │                    │                      │                      │
   │ POST /api/tasks    │                      │                      │
   │───────────────────▶│                      │                      │
   │                    │ 校验+落盘任务索引     │                      │ data/tasks/index.json
   │                    │ spawn dsh --profile P│                      │
   │                    │─────────────────────▶│                      │
   │                    │                      │ 文献调研→实验→分析    │
   │                    │                      │ (工具调用全量日志)    │ data/tasks/{id}.log
   │                    │                      │ 产出结构化 JSON       │
   │                    │◀─── 进程退出 ────────│                      │
   │                    │ 提取 JSON→结果落盘   │                      │ data/results/{id}.json
   │                    │ 记账（预算账本）      │                      │ data/ledger.json
   │ WS 日志流          │                      │                      │
   │◀───────────────────│                      │                      │
   │ GET /result        │                      │                      │
   │───────────────────▶│── 读取结果 JSON ────▶│                      │
   │◀─ 彩色渲染结果 ────│                      │                      │
└───────────────────────────────────────────────────────────────────────────────┘
全程可审计：任何一步都有日志/结果/账本可查；评测在 CI 定期回归整条链路
```

## 4.1 立项：为什么"非目标"清单最重要

科研 Agent 的三大场景：**文献调研、实验运行、数据分析**。每个都写进需求文档，但真正决定项目成败的是**非目标**：

```markdown
非目标（明确不做）：
- 不做实验设计决策（Agent 提方案，人拍板）
- 不自动部署/训练大模型（只编排已有脚本）
- 不联网乱跑代码（沙箱内运行，高危操作审批）
- 不做论文代写（只辅助片段与图表）
```

**为什么"不做什么"比"做什么"重要？** 因为 Agent 的不可控性主要来自范围模糊。把非目标写清楚，预设里就**不给它那些工具**——没有工具 = 做不了，这是比任何提示词都硬的控制（原理见 4.3）。

科研场景还有一个特殊风险：**Agent 会运行实验命令**——可能吃满 GPU、写爆磁盘、误删数据。所以 4.5 的沙箱/审批对科研比编码更重要。

## 4.2 环境：Ubuntu 主机的初始化要点

三个要点，每个都有原理：

1. **专用低权限账号**（`useradd -m researcher`）——Agent 进程的权限边界从"以谁的身份跑"开始；
2. **Node 22+（nvm）**——DSH 的运行时要求；
3. **配置分层**——服务器全局（`$DSH_HOME/cordis.patch.yml`）放通用项，项目（`/opt/autoresearcher`）放本项目，**API key 一律环境变量注入**（`.env` 不入库、systemd EnvironmentFile 管理）。

## 4.3 自定义预设：Agent 的能力面 = 一个文件

预设 = 一份 `agent.cordis.yml`（DSH 的 agent-plane 组合）。**它是产品的"能力面"：这个 Agent 能碰什么、不能碰什么，答案全在这一个文件里**。核心结构：

```yaml
- id: persona              # 人设：它是"审查者/调研者"，不是"改代码的人"
  name: '@deepseek-ai/dsh-persona'
  config:
    text: >-
      You are AutoResearcher... You NEVER design experiments autonomously.
      Critical operations (running experiments) require approval.

- id: tool-fs              # 只读能力
  name: '@deepseek-ai/dsh-tool-fs'

- id: research-tools       # 科研业务工具（我们的插件，4.4）
  name: '@autoresearcher/plugin-research'

- id: planning             # 计划模式：先评审后动手
  name: cordis:group
  group: true
  isolate: { planMode: true }
  config: [ { id: plan-mode, name: '@deepseek-ai/dsh-plan-mode' } ]

# 注意：故意【不】注册写工具（write/edit）与自由 shell
# —— 没有工具 = 做不了 = 最强的范围控制
```

> **为什么"不给工具"是最强控制**：模型只能通过你注册的工具与外界交互（第 1 篇 §3.1 的本质认知）。注册表里没有写工具，模型就物理上无法写文件——比任何"提醒它不要写"都可靠。企业级预设的设计顺序是：先列"绝不能做什么"，再列能力。

## 4.4 自定义工具：契约是灵魂，实现是细节

工具是 Agent 与世界的唯一接口，所以**契约（模型看到的 schema + 返回结构）比实现更重要**。以最危险的工具 `run_experiment` 为例，看契约的三层设计：

```ts
// plugin/src/index.ts（节选：run_experiment 的注册）
ctx.tools.register({
  name: 'run_experiment',
  description: '在沙箱内运行白名单实验脚本（data/scripts/ 下），返回退出码/日志尾部。',
  parameters: {              // ← 契约层 1：模型能传什么
    type: 'object', additionalProperties: false,
    properties: {
      script: { type: 'string', description: '相对路径，如 data/scripts/train.sh' },
      timeoutSec: { type: 'integer', default: 600 },
    },
    required: ['script'],
  },
  output: {                  // ← 契约层 2：模型能拿到什么（规范输出）
    schema: { type: 'object', additionalProperties: false,
      properties: {
        exitCode: { type: 'integer', required: true },
        tail: { type: 'string', required: true },
        logPath: { type: 'string', required: true },
        timedOut: { type: 'boolean', required: true },
      } },
  },
  async execute(args) {      // ← 实现层：内部细节，模型不可见
    return runExperimentScript(args.script, args.args ?? [], args.timeoutSec ?? 600)
  },
})
```

**为什么三层设计**：契约层 1 决定"模型会不会乱传参"（`additionalProperties: false` + 描述）；契约层 2 决定"下游（前端/评测/CI）能否类型安全地消费"；实现层可以随时重写而不影响任何调用方。

`run_experiment` 实现的三条铁律（科研安全的核心，全部在项目里）：

```ts
// plugin/src/experiment.ts（节选：安全三铁律）
// 铁律 1：路径必须解析后仍落在白名单根内（防 ../ 逃逸）
if (relative(SCRIPTS_ROOT, full).startsWith('..')) throw new Error('脚本必须在 data/scripts/ 下')

// 铁律 2：强制超时 + SIGKILL 进程树（防实验失控）
const timer = setTimeout(() => child.kill('SIGKILL'), timeoutSec * 1000)

// 铁律 3：日志全量落盘（审计 + 可复现）
writeFileSync(logPath, Buffer.concat(chunks).toString('utf8'))
```

> **为什么这些铁律长这样**：路径白名单防"模型被诱导运行任意命令"（提示词注入的载体）；超时防"实验吃满 GPU 拖垮主机"；日志落盘让"这台 Agent 跑过什么"可回答（审计，见 4.5）。每个安全机制都对应一个真实事故场景——设计安全时先列事故清单。

工具测试同理：**测边界，不测主路径**（空输入、超大输入、超时、路径逃逸、网络失败）——模型会以你意想不到的方式调用工具，边界行为就是真实行为。

## 4.5 安全与治理：先列事故清单，再设计防线

企业级安全的正确顺序：**先列"最坏会发生什么"，再逐条设计防线**。科研 Agent 的事故清单：

| 事故 | 防线 |
|---|---|
| 模型读走凭据（.ssh/.aws/.env） | fs 策略 deny 凭据目录 |
| 模型被诱导执行任意命令 | 脚本白名单 + 高危命令拦截 |
| 模型删数据 | 不注册删除类工具 + 沙箱只写 data/ |
| 实验失控吃满 GPU | 强制超时（SIGKILL）+ 资源熔断 |
| 越权操作 | 审批栈：**拒绝即终局**（不得换方式绕过） |
| 出事说不清 | 审计：工具调用全量落盘 + 实验日志全量留存 |

**"拒绝即终局"为什么重要**：如果允许"换个方式重试"，模型可以无限试探权限边界——这正是提示词注入攻击的核心手法。终局语义把试探成本变成零收益。

## 4.6 可观测性与成本：两个数字，两个机制

**可观测三件套**：日志（工具调用参数摘要与状态）、指标（token/时长/失败率按任务聚合）、追踪（一次任务 = 一条 trace）。DSH 的事件流天然支持这三者。

**成本控制的两个杠杆**（按收益排序）：

1. **上下文裁剪是最大的杠杆**：文献任务先检索摘要、只精读相关论文的摘要+结论页——省下的 token 比任何优化都多；
2. **模板化实验脚本**：Agent 只填参数不写新脚本——既省 token 又更安全（新脚本 = 新风险面）。

> 企业铁律：**没有成本上限的 Agent 项目，会在第一个月被账单教做人**。评测里固定断言"单任务输入 token ≤ N"，超了算回归失败。

## 4.7 评测体系：没有评测的 Agent 等于没有测试的软件

两层评测，缺一不可：

- **工具级单测**（4.4）：管"工具的正确性"——边界、失败路径、安全约束；
- **端到端评测**：管"任务完成度"——让 Agent 跑真实任务，用 grader 检查输出。

端到端评测的核心是 **manifest 驱动**：评测集是数据（case 列表），跑批是通用代码：

```jsonc
// evals/manifest.json（示意：一个用例）
{
  "id": "lit-kv-cache-survey",
  "task": "检索 3 篇关于 KV cache 优化的论文，输出标题与链接",
  "graders": [                                    // ← 评分函数：声明式
    { "type": "json-path", "path": "$.papers[0].link", "mustMatch": "arxiv.org" },
    { "type": "count", "path": "$.papers", "min": 3 }
  ],
  "budget": { "maxInputTokens": 15000 }           // ← 成本也是指标
}
```

跑批脚本只需 ~30 行（读 manifest → 逐个跑 headless 任务 → 按 grader 评分 → 汇总门禁退出码），核心逻辑就一句话：**通过率 ≥ 90% 且预算合规才 exit 0**。

**评测集管理四条规范**：版本化（fixture 不可变）、防饱和（模型升级后定期加新样例）、进 CI（预设/工具/提示词/模型改动必须跑）、双指标（正确性 × 成本同时门禁）。

> **为什么"模型升级必须跑评测"**：模型供应商静默升级模型后，同一提示词的输出分布会变——可能悄悄变笨。评测是唯一能察觉"悄悄变笨"的机制。这是 2026 年生产 Agent 团队踩坑最多的点。

## 4.8 部署：CI 门禁 + Ubuntu 托管

**CI 流水线**（GitHub Actions）分两个 job：`test`（单测/构建/冒烟，每次 push 跑）和 `eval`（评测门禁，tag 或手动触发，需 API key）。核心思路一句话：**代码改动过 test，能力改动过 eval，发布过两者**。

**Ubuntu 托管两方案**（原理：进程生命周期管理）：

```yaml
# 方案 A：Docker Compose（推荐）—— 后端镜像 + 前端静态站
services:
  backend:  { build: ./backend, ports: ["8000:8000"], restart: unless-stopped }
  frontend: { image: nginx:alpine, volumes: ["./frontend/dist:/usr/share/nginx/html:ro"] }
```

```ini
# 方案 B：systemd —— 免 Docker 的轻量选择
[Service]
User=researcher
EnvironmentFile=/opt/autoresearcher/.env    # API key 不入库
ExecStart=/usr/bin/python3 backend/server.py
Restart=on-failure
```

**为什么 Docker 优先**：镜像 = 版本化的部署单元（可回滚）；`restart: unless-stopped` = 崩溃自愈；volume 挂载 = 数据持久化与镜像解耦。systemd 适合单机轻量场景。

## 4.9 前后端分离：契约表 + 两个核心片段

前后端分离的验收标准：**停掉前端，用 curl 直接调后端 API 应该完全可用**——前端只是 API 的一个消费者。

**API 契约表**（前后端共同遵守，这就是"分离"的实质）：

```text
POST   /api/tasks              创建任务    req: {task, profile?} → {taskId, status}
GET    /api/tasks              任务列表    → [{taskId, status, createdAt}]
GET    /api/tasks/{id}         状态+日志   → {taskId, status, logTail}
GET    /api/tasks/{id}/result  结构化结果  → {report}（前端/评测消费）
WS     /api/ws/{id}            日志流      msg: {type: log|status|ping}
GET    /api/health             健康检查    → {ok, version}
# 鉴权：所有请求头 Authorization: Bearer <token>
```

后端核心（30 行，突出三个决策——鉴权、子进程隔离、结果收集）：

```python
# backend/server.py（节选：任务创建 = 三个决策）
@app.post("/api/tasks", dependencies=[Depends(auth)])   # 决策1：统一鉴权
def create_task(req: TaskRequest):
    task_id = uuid.uuid4().hex[:8]
    log_path = TASK_DIR / f"{task_id}.log"
    proc = subprocess.Popen(                            # 决策2：Agent 是子进程
        [DSH_BIN, "--profile", req.profile or "headless", req.task],
        stdout=open(log_path, "w"), stderr=subprocess.STDOUT)
    tasks[task_id] = {"status": "running", "proc": proc, "createdAt": time.time()}
    threading.Thread(target=_collect_result,            # 决策3：退出后收结果
                     args=(task_id, proc, log_path), daemon=True).start()
    return {"taskId": task_id, "status": "running"}
```

前端核心（25 行，突出两个决策——列表轮询 + 日志 WebSocket）：

```tsx
// frontend/src/App.tsx（节选）
const refresh = () => fetch('/api/tasks', { headers }).then(r => r.json()).then(setTasks)
useEffect(() => { refresh(); const t = setInterval(refresh, 5000); return () => clearInterval(t) }, [])

// 日志流：打开任务时订阅 WebSocket（断线自动重连，指数退避）
const ws = new WebSocket('ws://' + location.host + '/api/ws/' + currentId + '?token=' + TOKEN)
ws.onmessage = (e) => setLogs(l => [...l, JSON.parse(e.data).data])
```

> **为什么轮询 + WebSocket 组合**：任务列表低频变化用轮询（简单可靠），日志流高频变化用 WebSocket（实时）。每个技术选择都对应一个频率特征——这是前后端交互设计的通用方法论。

## 检验：读完本节，你能回答吗

1. 为什么“不给工具”是比任何提示词都硬的范围控制？（原理是什么）
2. 安全设计的正确顺序是什么？为什么先列事故清单？
3. “拒绝即终局”为什么重要？它防的是什么攻击手法？
4. 为什么模型升级必须跑评测？“悄悄变笨”为什么只能靠评测发现？
5. 前后端分离的验收标准是什么？“分离”的实质（契约表）是什么？

## 本章小结（九条心法，条条是原理）

1. **架构先行**：前端只是 API 消费者——"分离"靠契约表维持，不靠代码结构；
2. **非目标优先**：范围模糊是 Agent 失控的根源；
3. **不给工具 = 最强控制**：注册表是唯一交互面；
4. **契约是灵魂**：schema 决定模型行为，输出契约决定下游安全；
5. **先列事故清单再设计安全**：每条防线对应一个事故场景；
6. **拒绝即终局**：防权限试探；
7. **成本是硬约束**：上下文裁剪是第一杠杆；
8. **评测防"悄悄变笨"**：模型升级必须跑回归；
9. **部署 = 生命周期管理**：镜像版本化、崩溃自愈、数据解耦。

**完整工程代码**（不是本节的重点，但都在）：`../autoresearcher/`（backend/plugin/frontend/evals/deploy + 文档四件套），全部通过真实测试（pytest 10/10、vitest 22/22、tsc 0 error、冒烟 5/5）。

**下一篇**：[第 5 篇 · 前沿方向与学习路线](05-前沿方向与学习路线.md)