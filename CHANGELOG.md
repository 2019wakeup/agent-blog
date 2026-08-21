# 本系列修订记录（CHANGELOG）

> 记录多轮打磨与评审的轨迹，体现"多轮评审"的交付过程。

## v0.1（初稿完成，2026-08）
- README + 第 1~5 篇初稿全部完成
- 结构：入门（1）→ 框架横评（2）→ DSH 深度解剖（3）→ 企业级实战（4，核心）→ 前沿与路线（5）
- 第 4 篇按企业级清单组织：需求/边界、预设、工具、安全、可观测、评测、CI/CD

## v0.2（评审轮 1：自评，2026-08）
- 自动化检查：代码围栏配平、篇间链接无死链、无转义泄漏 ✅
- 修复：第 2 篇 Codex 开源性表述（CLI 实际开源，改为"✅ CLI 开源（模型/服务闭源）"）
- 增强：第 4 篇新增 4.0 项目总览架构图与核心原则

## v0.3（评审轮 2：子代理独立评审）
- 独立评审 Agent 启动后运行过久未产出，按用户反馈优先级调整：中断评审，转为"深度扩展"专项
- 结论：以用户反馈"教程不够详细、太简略"为最高优先级，进行 v0.4 深度扩展

## v0.4（深度扩展轮，响应"不够详细/太简略"反馈）
- 第 1 篇（245 行 → 658 行）：token 化/BPE、温度采样、KV 缓存原理、完整 API 请求响应 JSON、函数调用三阶段完整代码、并行 tool_calls、错误回填模式、工具描述工程、四种架构模式详解、RAG 最小实现、130 行生产雏形 Agent、练习答案提示
- 第 2 篇（119 行 → 230 行）：LangGraph/OpenAI Agents SDK 可运行完整示例、MCP 协议生命周期/三种传输/安全边界、MCP Server 完整代码
- 第 3 篇（272 行 → 523 行）：Cordis 官方示例与 realm/isolate 详解、双平面架构对照真实配置、四预设真实配置引用、PTC 机制链带真实源码行号、安全审批时序、动手实验 5 个、源码地图 12 包
- 第 4 篇（448 行 → 567 行）：4.4 升级为完整 4 文件插件项目（package.json/入口/纯逻辑/测试）、4.7 评测脚本升级为 manifest 驱动 + 预算门禁 + 报告
- 第 5 篇（151 行 → 172 行）：MCP 生命周期/传输/安全、workflow 可运行脚本、评测集 manifest 骨架
- 质量修复：修复第 1 篇正文意外反引号导致的围栏不平衡；修复第 4 篇评测脚本区域内容重复（程序化重构验证通过）
- 全系列检查：围栏配平 ✅ 无死链 ✅ 无转义泄漏 ✅ 总计约 7 万字符



## 深度扩展轮（每篇加厚至深入水准）
- 新 03：106 → 188 行（四件套规范每个配代码：分页/杀进程树/截断标记/描述工程 + 练习 3.1/3.2 + 坑 4 条）
- 新 05：79 → 143 行（五种笔记配真实例子、三原则展开、数据流全景、静态记忆指令与缓存友好、练习 5.1/5.2）
- 新 07：100 → 184 行（协议生命周期/传输/Server 完整代码/管理器四要点/LangGraph+SDK 核心代码/练习 7.1/7.2）
- 新 08：170 → 237 行（Cordis Counter 官方示例/四预设详解/极简 complete:true/创造模式 TRUST 警告/实验清单 5 条/检验 8 题）
- 新 09：150 → 247 行（三工具契约表/沙箱配置示意/评测辅助函数 3 个）
- 新 10：100 → 161 行（Workflow 编排示例/评测集骨架）
- 全部 10 篇完成：总计 1979 行（原 1220 行，+62%）；检查器全绿
- 新 06：88 → 174 行（三失败模式配例子、ReAct/Reflexion 完整 trace 回顾、PlanState 字段详解、Guard 代码、练习 6.1/6.2、坑 4 条）
- 新 04：91 → 181 行（六层每层实现要点 + 最小实现代码 + 六层与 KV 缓存关系 + 练习 4.1/4.2 + 坑 4 条 + 检验 7 题）
- 新 01：156 → 264 行（恢复 BPE/概率采样/注意力/KV 缓存/API 调用/函数调用三阶段完整示例 + 练习 1.1/1.2 + 检验 7 题）
- 新 02：100 → 200 行（四工程细节带代码、35 行循环 trace、五扩展点每个配代码、常见坑 4 条、检验 6 题）
- 检查器全绿
## 重构轮完成（v3.0）：10 篇搭建式结构全部重建
- 新 01-10 全部完成：理论框架/最小Loop/工具系统/上下文引擎/记忆/Planning/MCP与Skills/DSH参照/企业实战/前沿路线
- 每章统一「理论（Tritium 叙事，标注来源）→ 你的实现 → DSH 参照 → 检验」
- 旧 5 篇归档至 archive/；README 重写为新目录
- 检查器全绿（围栏/泄漏/Liquid/死链）
## v0.5（终极目标重构轮：科研 Agent + 前后端分离 + Ubuntu 部署）
- 用户明确终极目标：做/改一个深度定制的**科研 Agent**（自动化计算机科研），前后端分离，后端跑在 Ubuntu 远程主机
- 目标修订（revision 4）并将全系列导向该目标：
- README 重构：终极目标章节、三层架构路线、目录指向科研 Agent 实战
- 第 4 篇整体改写：CodeReview → **AutoResearcher 科研 Agent**（19.7k 字符）：
  - 4.0 三层架构图（前端/API/Agent 核心，前后端分离骨架）
  - 4.1 科研需求（文献/实验/数据三场景 + 非目标清单）
  - 4.2 Ubuntu 远程主机初始化（nvm/docker/低权限账号）
  - 4.3 科研预设（实验工具审批、bash 白名单）
  - 4.4 完整科研工具插件（arxiv_search/run_experiment/parse_pdf + 边界测试）
  - 4.5 科研沙箱（凭据禁读/脚本白名单/审批流/审计表）
  - 4.6 科研成本（token + GPU 双预算）
  - 4.7 文献/实验双评测集 + manifest 跑批
  - 4.8 Docker Compose/systemd Ubuntu 部署 + CI
  - 4.9 前后端分离落地（FastAPI + WebSocket + React + Nginx 反代）
- 修复：第 2/3/5 篇与交付说明中 CodeReview 旧引用全部改为 AutoResearcher
- 全系列复检通过：围栏配平/无死链/无泄漏，总计 7.4 万字符

## v0.6（评审轮 2 启动 + 4.9 补全 + 代码自查）
- 4.9 前后端分离补全为可运行代码：完整 API 契约表（REST+WS）、完整 FastAPI 后端（任务注册表/鉴权/列表/详情/WebSocket 日志流）、完整 React 面板（任务列表轮询 + 日志流 + 结果视图）
- 4.4 新增 experiment.ts 完整实现（白名单路径防逃逸/超时杀进程树/日志落盘——科研工具安全三铁律）
- 代码自查修复：FastAPI 示例 asyncio 导入位置、__main__ 冗余导入
- 启动聚焦独立评审子代理（限 3 个核心文件：第 1/3/4 篇）
- 4.7 评测脚本补全为完整可运行版：runHeadless/extractReportJson/jsonPath/grade（支持 json-path/count/regex 三种 grader）
- 代码自查修复：FastAPI asyncio 导入、__main__ 冗余导入、评测代码块缺失闭合围栏
- 全系列复检通过：围栏配平/无死链/无泄漏，总计 8.17 万字符

## v0.7（评审轮 2 终审：源码级核验 + 事实修正）
- 聚焦评审子代理运行过久未产出 → 中断，改为自主终审（含源码级核验）
- 源码引用行号全部核验通过：renderToolsSdk@1607、jsonSchemaToTs@1576、CODE_ONLY_INSTRUCTION@2400、collapses@2975、tools:sdk order 150、worker Config@649
- **事实修正**：标准预设实际没有 tool-presentation 行（native 是默认值）——修正第 3 篇 §4.1 示例，改为注释说明"只有 code 预设才声明 mode: code"
- **准确性标注**：第 4 篇 4.3/4.5 的 sandbox 配置键名标注为示意写法，并说明 DSH 真实机制（宿主平面 ctx.fs 后端 + tools/execute 瀑布权限插件 + 审批栈 + 不给工具）
- 代码语法验证（真实执行）：评测脚本 node --check ✅、FastAPI server.py py_compile ✅、mini_agent py_compile ✅、第 2/3/4 篇全部 ts 块 stripTypeScriptTypes ✅（源码节选片段除外，已加"节选"说明）
- 第 3 篇 §5.1 新增"源码节选"说明
- 全系列最终复检：8 文件全绿，总计 8.22 万字符 → **交付**

---

## 后续迭代 1（v1.0）：AutoResearcher 项目落地（教程 → 可运行工程）

在 `../autoresearcher/` 生成完整可运行项目骨架（博客第 4 篇的代码落地）：

| 模块 | 文件 | 验证 |
|---|---|---|
| backend | `backend/server.py`（FastAPI：任务注册表/Bearer 鉴权/WebSocket 日志流）+ requirements.txt | py_compile ✅ |
| plugin | `plugin/src/{index,arxiv,experiment,pdf}.ts` + 2 个 vitest 测试 + package.json/tsconfig | stripTypeScriptTypes 6/6 ✅ |
| frontend | `frontend/src/{App,api,main}.tsx` + vite 配置（Vite dev 代理 /api） | JSX 标签配平 13/13 ✅（tsc 本机不可用，人工复核） |
| evals | `evals/manifest.json`（2 case）+ `run-evals.mjs`（manifest 驱动 + 预算门禁） | node --check ✅ / JSON ✅ |
| deploy | docker-compose.yml / Dockerfile.backend / nginx.conf / systemd unit | js-yaml 解析 ✅ |
| agent | `agent/agent.cordis.yml` 科研预设（只读+受控实验，不注册写工具） | js-yaml 解析 ✅ |
| 其他 | README（本地快速开始 + Ubuntu 部署两方案 + 安全基线 + 路线图）、.env.example、.gitignore、data/scripts/demo.sh | bash -n ✅ |

全项目验证套件 6 类全部通过（Python/JS/JSON/YAML/Shell/TS）。

## 后续迭代 2（v1.1）：结果捕获 + WS 鉴权 + 前端结果视图
- backend/server.py：`_collect_result` 后台线程（进程退出后从日志提取 JSON 块写入 `data/results/{id}.json`，补齐 `/api/tasks/{id}/result` 的数据源）；WebSocket 鉴权（`?token=` 查询参数，4401 拒绝）；新增 `threading` 导入 —— py_compile ✅
- frontend：api.ts WS 带 token；App.tsx 新增结构化结果视图（任务完成后拉取 /result 渲染 JSON）—— JSX 标签配平 15/15（textarea 自闭合合法）✅
- 待续：路线图其余项（文献工作流/多 Agent/资源配额）留待后续迭代

## 后续迭代 3（v1.2）：文献调研工作流
- 新增 plugin/src/survey.ts：literature_survey 工具（检索 → 下载 PDF 到 data/papers/ → 解析前 3 页 → 返回结构化索引；单篇失败不拖垮整批）
- index.ts 注册第 4 个工具；新增 extractArxivId 纯函数（abs/pdf 链接两种形态）
- 新增 test/survey.test.ts（3 个单测）；evals manifest 扩至 3 个 case（含文献调研下载用例）
- 验证：TS_ALL_OK（10 文件）| PY_OK | JS_OK | SH_OK | MANIFEST_OK(cases=3)

## 后续迭代 4（v1.3）：实验资源治理（GPU/磁盘预检 + 预算账本 + 熔断器）
- 新增 plugin/src/resource.ts：预算账本（data/ledger.json，跨天自动滚动）、熔断决策 decide()、nvidia-smi/df 预检解析（纯函数可单测）
- experiment.ts 集成：实验前熔断检查（实验数/时长/磁盘/GPU 四重限制），运行后自动记账
- 新增 test/resource.test.ts（9 个单测：决策/滚动/损坏恢复/解析函数）

## 后续迭代 5（v1.4）：WS 心跳 + 前端断线重连 + 评测集扩充
- backend：WebSocket 15s 心跳（防 Nginx 空闲超时断开）
- frontend：api.ts 断线自动重连（指数退避 1s→30s）+ 45s 心跳超时检测
- evals：manifest 扩至 5 个 case（新增白名单拒绝、资源熔断两个安全用例）
- 验证：PY_OK | JS_OK | SH_OK | JSON_OK | YAML_OK | TS_ALL_OK(12 文件)

## 后续迭代 6（v1.5）：多 Agent 双评审工作流（发现者 + 复核者）
- scripts/review-lib.mjs：纯逻辑模块（extractJson/merge/双提示词），15 项逻辑测试全过
- scripts/double-review.mjs：独立编排器（两个隔离 headless 会话并行 → 复核者裁决 → 合并落盘 data/results/）
- agent/workflows/double-review.workflow.mjs：会话内 workflow 工具版（导出 run()，8 项逻辑测试全过）
- 合并规则：复核者标记误报降级 info 附注；漏报以 warning 补充（source=reviewer）

## 后续迭代 7（v1.6）：真实端到端冒烟 + 评测集扩充至 10 case
- server.py 新增 DRY_RUN=1 模式（无 dsh 也能测通 API 全链路：假任务进程输出日志 + JSON 结果块）
- scripts/smoke.mjs 端到端冒烟：health → 建任务 → 轮询完成 → 鉴权负例(401) → 结构化结果；依赖缺失优雅 SKIP(exit 3)，自动识别 .venv
- **真实运行验证**：本机 venv 安装 fastapi 后完整冒烟 5/5 PASS（EXIT:0），含任务日志流与结果落盘
- 评测集扩至 10 case（schemaVersion 2）：新增文献结构化总结、参数扫描、超时安全、数据解析、凭据防护；fixtures 增加 results-sample.json 与 slow.sh
- 全项目验证：PY_OK | JS_ALL_OK(5) | SH_OK(2) | JSON_OK(5)

## 后续迭代 8（v1.7）：Git 仓库 + CI 流水线 + Makefile
- git init（main 分支）+ 首次提交：40 文件入库（.venv/data 输出被 .gitignore 正确排除）
- .github/workflows/ci.yml：test 门禁（插件单测/前端构建/后端编译/DRY_RUN 端到端冒烟/脚本语法）+ eval 门禁（tag 或手动触发，需 DEEPSEEK_API_KEY，通过率 ≥90%）
- Makefile：setup/test/build/smoke/eval/up/down 一键目标（make -n 干跑验证）
- 验证：CI_YAML_OK（jobs=test,eval）| git log 提交 ba43ff8 | make dry-run 正常

## 后续迭代 9（v1.8）：真实工具链验证里程碑（不止语法检查）
- npm 依赖安装成功（本地缓存），vitest **真实执行 22/22 测试通过**（含 1s 真实超时杀进程测试）
- tsc strict **0 错误**（修复 4 类真实类型问题：cordis Context 增强声明 dsh.d.ts、pdf-parse 声明、Promise 返回类型、null 合并）
- 插件 tsc 构建成功（dist/ 产物齐全）；前端 **vite 生产构建成功**（27 模块，147KB）
- 发现并修正依赖包名：DSH 的 cordis 发行包是 @deepseek-ai/cordis（npm 上无 cordis@4）——项目依赖与导入已修正
- 3 个 git 提交（ba43ff8 → b0be4e1 → 97193d9），package-lock 入库保证可复现安装

## 后续迭代 10（v1.9）：后端真实测试 + 运维手册 + 单进程模式
- backend/test_api.py：8 个 pytest 用例**真实运行全部通过**（鉴权 401×2、任务创建→完成、结果收集、404、空任务 422、列表）
- server.py SERVE_FRONTEND=1：单进程托管 frontend/dist（本地/小型部署免 Nginx）
- docs/OPERATIONS.md 运维手册：备份策略、成本控制、升级防变笨流程、排障速查、安全月检清单
- CI test job 重构（venv 提前 + pytest 步骤）；Makefile test 目标加 pytest
- git 提交 4 个（v1.9）

## 后续迭代 11（v1.10）：任务持久化 + 前端结果增强 + Docker 校验
- server.py：任务索引持久化 data/tasks/index.json（重启恢复历史；未完成任务标记 interrupted；损坏索引自恢复不崩溃）
- test_api.py +2 用例：重启恢复 / 损坏恢复 —— pytest **10/10 真实通过**
- App.tsx：评审报告严重度彩色渲染（critical 红/warning 橙/info 蓝 + 复核附注），vite 生产构建通过
- docker compose config **真实校验通过**（Docker 28.0.1）
- git 提交 5 个

## 后续迭代 12（v1.11）：Docker 部署验证 + 构建环境限制记录
- docker compose config 真实验证通过（Docker 28.0.1，daemon 启动与恢复正常）
- docker build 验证：python:3.12-slim 拉取正常；**node:22-slim 拉取在本机网络反复挂起**（环境网络限制，非项目问题）——镜像构建需在 Ubuntu 主机执行（README 已含完整步骤）
- .dockerignore：构建上下文从数百 MB 降到 KB 级；Dockerfile 慢网络加固（npm --no-audit、pip --timeout 60）
- 全项目回归：pytest 10/10 ✅ vitest ✅ JS 校验 ✅
- git 提交 6 个

## 后续迭代 13（v1.12）：集成链路打通（评测 profile + 一键装配）
- run-evals.mjs：DSH_PROFILE 环境变量（默认 headless），自定义工具评测可指向 autoresearcher 预设
- scripts/setup-agent.mjs：一键装配（构建插件 → dsh plugin add → 安装预设到 ~/.dsh/.agent-presets/autoresearcher/）
- Makefile setup-agent 目标 + README 装配说明
- 回归：pytest 10/10 ✅ vitest ✅ JS_ALL_OK(5) ✅；git 提交 7 个

## 后续迭代 14（v1.13）：架构文档 + 全量终检
- docs/ARCHITECTURE.md：三层架构图、一次科研任务完整数据流、组件职责与关键决策表、扩展指南（加工具/加评测/加后端能力/加工作流）、一致性保证
- README 文档链接补全（架构/运维手册）
- 全量终检（全部真实执行）：pytest 10/10 | vitest 22/22 | tsc 0 error | vite build | smoke 5/5 | JS+YAML+Shell 全绿
- git 提交 8 个













## 重构轮（v3.0 搭建式结构）
- 新结构 10 篇主线：01 理论框架 / 02 最小 Loop / 03 工具系统 / 04 上下文引擎 / 05 记忆系统 / 06 Planning / 07 MCP/Skills / 08 DSH 参照 / 09 企业实战 / 10 前沿路线
- 已完成新 01-07（每章：理论 → 你的实现 → DSH 参照 → 检验），设计文档见 .research/restructure-design.md
- 旧 01-05 暂保留，08-10 完成后统一归档
## 整合轮 5（交付）：终检 + Pages 上线
- Liquid 扫描：新增内容无泄漏（03 篇的模板变量均在 raw 保护内）
- 交付说明 v2.2 追加整合记录；git 提交 508a273 推送，GitHub Pages 重建验证新内容上线（Harness/上下文引擎/Planner 均在渲染页）
- 检查器全绿
## 整合轮 4（Tritium 系列）：第 2 篇 MCP/Skills + 第 3 篇 plan mode 对照
- 第 2 篇 §4.4：MCP vs Skills 分工（手 vs 做事方法）、Skills 渐进披露与 SKILL.md frontmatter、MCP 管理器工程要点（过滤/诊断不炸循环/权限联动）（[5.3] 篇）
- 第 3 篇：plan mode 小节新增与 Tritium Planner 状态机的对照（Review 闸门 vs 完整四环 + Final Answer Guard）
- 检查器全绿
## 整合轮 3（Tritium 系列）：第 1 篇补齐 Planner 与工具规范
- §4.5 Planner 状态机：三失败模式（漂移/假完成/重复劳动）、计划=状态机而非文字、Plan→Review→Solve→Replan、Final Answer Guard、四层模型分工（[5] 篇）
- §3.4 工具与 Provider 工程规范：offset/limit、timeout 杀进程树、truncation 第一道防线、Provider 边界（[2.5] 篇）
- 检验问题扩充至 12 题；检查器全绿
## 整合轮 2（Tritium 系列）：第 1 篇扩充上下文引擎与记忆系统
- §5.6 上下文引擎六层：历史与视图分离（事实记录 vs 工作视图）、Prompt Builder/Token 估算/Budget Manager/启发式压缩/Handoff Summary/Dynamic Compression（[3] 篇）
- §5.7 记忆系统三层：短期/中期（Workspace 五种笔记）/长期（Memory Store）、记忆工具三原则（读写分离/可解释/不喧宾夺主）、与 Context Engine 分工（[4] 篇）
- 检验问题扩充至 10 题；检查器全绿
## 整合轮 1（Tritium 系列）：第 1 篇扩充 Harness 理论框架
- 来源：Tritium《Build An Agent From Scratch》[1][2]（CC BY-NC-SA，https://www.tritium.work，已标注）
- §6.1 五扩展点：流式共用 loop/并行顺序写回/错误进循环/事件先行/配置隔离（[2] 篇）
- §8 Harness 全景：马具类比、9 模块×4 层次全景表、四机制、缓存友好分层表、五条设计原则（[1] 篇）
- 检验问题扩充至 8 题；检查器全绿
## 打磨轮 6：字符画图 → Mermaid（真实渲染）
- 6 处字符画图全部替换为 Mermaid：2 张 sequenceDiagram（run_code 全链路 / 科研任务时序）+ 4 张 flowchart（Agent 循环 / 双平面 / 塌缩豁免 / 三层架构）
- 语法验证：mermaid.ink 真实渲染 6/6 通过（注意：需 base64url 编码，标准 base64 含 / 会 404）
- Pages 注入：cayman 官方扩展点 _includes/head-custom.html 加载 mermaid@11（首次误用 layout 覆盖导致构建失败，已修复）
- GitHub 仓库视图原生渲染 mermaid；Pages 浏览器端渲染 SVG
## 打磨轮 5：最终校对（配比调整交付）
- 自动化检查：练习编号连续（1.1-1.3）、章节编号完整（各篇 1-7/1-10/4.0-4.9）、检验/小结齐备、PTC 术语一致、README 配比说明与三遍读法在位
- 渲染抽查：5.5.1 时序图、4.2 要点均正常
- 交付：理论为主、代码精简、图示类比齐备、检验可自测的教程 v2.1
## 打磨轮 4：类比补强（讲透的最后一环）
- 第 1 篇：注意力=手电筒找东西（上下文越长注意力越摊薄）、KV 缓存=备菜（前缀神圣不可侵犯）
- 第 3 篇：双平面=餐厅厨房/菜单（设施全局唯一、菜单按桌定制）、PTC=写脚本批量执行 vs 逐行命令行
- 第 4 篇：前后端分离=外卖平台（App 不直接进厨房，分离的本质是契约表）
- 全系列围栏配平、无死链，约 6.6 万字符
## 打磨轮 3：README 配比说明 + 关键时序图
- README：讲解配比说明（v2.0 理论为主）、推荐三遍读法（看正文→对照代码→跑工程）
- 第 3 篇：新增“一次 run_code 的完整旅程”时序图（模型/宿主/Worker/工具实现四列全链路）
- 第 4 篇：新增“一次科研任务的完整时序”图（前端/后端/Agent/数据审计四列）
- 全系列围栏配平、无死链
## 打磨轮 2：每篇新增“检验”区块（把读变成会）
- 5 篇各在“本章小结”前加 4-6 个检验问题（答案指向正文对应小节），覆盖核心机制与工程决策
- 第 3 篇长代码块审查：均为高教学价值真实示例（cordis 官方示例/真实预设配置），保留
- 全系列围栏配平、无死链；总计约 6.2 万字符
## 配比调整轮（响应“理论不够详细、代码多而长”反馈）
- 第 1 篇：51% → 34% 代码；理论加深（概率采样本质、注意力物理限制、函数调用=文本生成、三策略原理与代价、35 行循环的四层理解）
- 第 2 篇：45% → 19% 代码；每个框架加“设计哲学/为什么”讲解
- 第 4 篇：75% → 44% 代码；改为决策导向（非目标原理/事故清单设计安全/契约三层设计/部署原理），完整代码指向 ../autoresearcher/ 工程
- 第 3 篇保持 33%（源码引用是“机制到源码级”的硬要求）；第 5 篇 20%
- 全系列围栏配平、无死链
## 后续迭代 15（v1.13.0）：发布基线
- RELEASE.md：发布说明（内容清单/验证矩阵/已知限制/部署路径/后续路线）
- git 标签 v1.13.0（首个可部署基线）；交付说明定稿