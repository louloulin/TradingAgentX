# TradingAgentX 代码库分析

> 本文面向希望二次开发、排查问题、扩展数据源或自托管部署的工程师。仓库目录名为 `TradingAgentX`，README 与对外页面主要使用 `TradingAgents-AShare` 作为产品名；下文统一把代码仓库称为 TradingAgentX，把对外能力称为 TradingAgents-AShare。

## 文档范围与阅读方式

本文按“总览 -> 引擎 -> 后端 -> 前端 -> 配置与部署 -> 审校整合”的顺序组织。第 1 章先回答三个基础问题：

1. 这个仓库解决什么问题。
2. 代码分成哪些层次。
3. 阅读和调试时应该先从哪里进入。

如果你想直接跟踪一次分析请求如何落地，建议先阅读本章的“关键执行链路”，再继续看[第 2 章](#chapter-2)和[第 3 章](#chapter-3)。

## 第1章：项目概览与仓库结构

### 1.1 项目定位

TradingAgentX 把多智能体投研框架、Web 控制台、开放 API 和第三方技能接入放进了同一个仓库。它要解决的不是“调用一次大模型生成结论”，而是把一条完整的投研流程产品化：

1. 接收自然语言或标准股票代码输入。
2. 识别标的、日期、周期和部分用户意图。
3. 拉取行情、新闻、基本面、资金流向等数据。
4. 组织多角色分析、辩论、交易决策和风控审查。
5. 输出结构化研报并保存为可查询的历史记录。

从 README、后端入口和技能定义来看，这个仓库当前主要面向 A 股分析场景，同时保留了对美股 ticker 的兼容能力。系统对外提供三种使用形态：

- Web 控制台：面向人类用户发起分析、查看进度、回看报告和管理配置。
- REST API：面向脚本、第三方系统或自动化任务提交分析请求并查询结果。
- 技能接入：面向 OpenClaw 一类 Agent 平台，把分析能力作为外部工具复用。

### 1.2 核心能力与系统边界

仓库当前公开暴露的核心能力可以归纳为五类：

- 多智能体分析：系统按“分析师 -> 博弈与研究 -> 交易 -> 风控 -> 组合裁决”的五阶段组织 15 个角色协作。
- 多源数据整合：核心数据层统一封装行情、新闻、基本面、资金与指标查询，屏蔽不同供应商差异。
- 双入口执行：既支持前端交互式使用，也支持标准化 API 调用和第三方技能触发。
- 结果持久化：分析任务状态、结构化指标和全文研报会写入数据库，供后续查询、报告页展示和历史追溯。
- 自托管部署：仓库同时包含 Python 后端、前端静态站点和 Docker 运行路径，适合独立部署。

它的边界也很明确：

- 它是“投研协作系统”，不是实时高频交易引擎。
- 它会生成结构化交易建议，但不会直接接券商下单。
- 它有回测页和报告管理能力，但回测与执行并不是多智能体主图的一部分。

### 1.3 技术栈与交付形态

从 `pyproject.toml`、`frontend/package.json` 和运行入口可以看出，TradingAgentX 采用的是典型的前后端分层结构：

| 层次 | 主要技术 | 作用 |
| --- | --- | --- |
| 核心分析引擎 | Python 3.10+、LangGraph、LangChain 兼容模型客户端 | 编排智能体、数据工具、辩论流程与状态传播 |
| 服务层 | FastAPI、Pydantic、SQLAlchemy、Uvicorn | 提供认证、分析任务、报告查询、配置管理与流式响应 |
| 前端控制台 | React 18、TypeScript、Vite、React Router、Zustand | 提供登录、分析、报告、设置、回测和组合页面 |
| 数据与持久化 | SQLite 默认库、JSON 字段、LangGraph SQLite checkpoint | 保存用户、令牌、报告结果与图执行检查点 |
| 交付与集成 | Docker、REST API、技能描述文件 | 支持容器部署、外部系统调用和 Agent 平台集成 |

这意味着仓库并不是“一个 Python 包外加几个脚本”，而是一个完整的产品仓库：`tradingagents/` 负责算法与编排，`api/` 负责对外服务，`frontend/` 负责交互层，三者在同一个代码库里共同演进。

### 1.4 关键执行链路

第 1 章只做全局说明，不展开每个节点的内部实现，但先给出一条请求从入口到结果的主路径：

1. 用户从前端页面、API 客户端或技能平台提交分析请求。
2. [`api/main.py`](../api/main.py) 负责认证、参数标准化、股票识别、任务登记和后台执行。
3. 后端创建 [`TradingAgentsGraph`](../tradingagents/graph/trading_graph.py) 实例，加载模型客户端、数据采集器、工具节点和图状态机。
4. `tradingagents/graph/` 调用 `agents/` 中的分析师、研究员、交易员与风控角色，按条件路由推进整条投研链。
5. `dataflows/` 为智能体提供行情、新闻、财报、指标和资金流数据，`llm_clients/` 负责不同模型提供商的接入。
6. 结果经后端序列化后写入 [`api/database.py`](../api/database.py) 定义的报告表，再通过任务查询接口和报告接口返回给前端。
7. 前端在 [`frontend/src/App.tsx`](../frontend/src/App.tsx) 定义的路由体系中展示控制台、分析页、历史报告、设置、回测和组合页面。

如果你需要在本地快速跑通这条链路，最小入口如下：

```bash
uv run python -m uvicorn api.main:app --port 8000
cd frontend && npm run dev
```

后端默认会初始化本地 SQLite 数据库，前端通过 Vite 开发服务器提供界面。完整配置项、环境变量和容器部署方式放在[第 5 章](#chapter-5)展开。

### 1.5 仓库目录结构

下表列出当前最值得关注的顶层目录与文件：

```text
.
├── api/                    FastAPI 服务入口、认证、任务编排与报告接口
├── assets/                 README 与前端使用的示意图、截图资源
├── docs/                   中文设计文档与本代码分析文档
├── frontend/               React + TypeScript 控制台
├── skills/                 第三方 Agent 平台技能定义
├── tests/                  Pytest 测试集
├── tradingagents/          Python 核心库
├── README.md               产品说明、快速启动和对外能力介绍
├── pyproject.toml          Python 包配置与依赖声明
├── requirements.txt        兼容性依赖清单
└── Dockerfile              一体化容器构建入口
```

更细一层的阅读重点如下：

| 路径 | 主要内容 | 为什么值得先读 |
| --- | --- | --- |
| `tradingagents/graph/` | LangGraph 图搭建、条件路由、数据采集、反思与信号处理 | 这里定义了“多智能体如何接力执行” |
| `tradingagents/agents/` | 各类分析师、研究员、经理人、交易员和风控角色 | 这里体现了系统的业务分工 |
| `tradingagents/dataflows/` | 数据供应商抽象、A 股与通用市场数据接口 | 这里决定了智能体拿到什么原始信息 |
| `tradingagents/llm_clients/` | OpenAI、Anthropic、Google 等模型客户端封装 | 这里决定了模型供应商如何替换 |
| `tradingagents/prompts/` | 中英文提示词目录与 prompt catalog | 这里承载角色行为约束与输出格式 |
| `api/` | Web API、鉴权、任务执行、报告读写 | 这是所有外部调用的第一入口 |
| `frontend/src/` | 页面、组件、状态管理、API 调用 | 这里决定用户如何看到分析过程和结果 |
| `tests/` | API smoke test、状态测试、数据采集测试、多周期图测试 | 这里能快速说明仓库真正关心哪些行为 |
| `skills/tradingagents-analysis/` | 外部技能描述与调用约定 | 这里展示仓库如何被其他 Agent 系统复用 |

### 1.6 推荐阅读顺序

如果你的目标是理解全局结构，推荐按下面顺序进入代码：

1. [`README.md`](../README.md)：先理解产品定位、支持的部署方式和对外能力。
2. [`api/main.py`](../api/main.py)：确认系统对外暴露了哪些请求入口，以及后端如何接入核心引擎。
3. [`tradingagents/graph/trading_graph.py`](../tradingagents/graph/trading_graph.py)：确认核心图对象如何组装模型、工具、记忆和状态机。
4. [`tradingagents/agents/utils/agent_states.py`](../tradingagents/agents/utils/agent_states.py)：确认图执行期间共享了哪些状态。
5. [`frontend/src/App.tsx`](../frontend/src/App.tsx)：确认前端有哪些一等页面和导航入口。
6. [`tests/test_api_smoke.py`](../tests/test_api_smoke.py)：确认仓库如何用可执行测试约束主要 API 行为。

这个顺序的好处是：先看入口，再看编排，再看状态与 UI，最后用测试反证自己的理解。

### 1.7 后续章节导览

本文后续章节会继续沿着“从总览进入细节”的顺序展开：

| 章节 | 关注点 |
| --- | --- |
| [第 2 章](#chapter-2) | 多智能体角色分工、LangGraph 结构、状态流转与主执行链 |
| [第 3 章](#chapter-3) | FastAPI 路由、任务生命周期、鉴权、数据库与报告存储 |
| [第 4 章](#chapter-4) | 页面结构、前端状态管理、接口消费与用户交互流 |
| [第 5 章](#chapter-5) | 数据供应商、模型配置、环境变量、启动方式和部署方案 |
| [第 6 章](#chapter-6) | 统一术语、交叉引用、缺口检查与总体验证 |

第 1 章的任务到这里就完成了：你现在应该已经知道仓库的产品定位、核心目录和建议阅读顺序。接下来进入[第 2 章](#chapter-2)，再把“目录结构”进一步拆成“执行机制”。

<a id="chapter-2"></a>
## 第2章：多智能体核心引擎与执行流程

第 2 章回答两个核心问题：这个项目的“多智能体”到底由哪些角色组成，以及一次请求进入后是如何在 LangGraph 里流转、汇总、回退并落成最终建议的。第 1 章已经说明仓库分层；本章进一步把 `tradingagents/graph/`、`tradingagents/agents/` 和 `tradingagents/prompts/` 拆成可执行的引擎视图。

### 2.1 引擎总装：`TradingAgentsGraph` 负责把零件组装成工作流

多智能体主引擎的门面类是 [`tradingagents/graph/trading_graph.py`](../tradingagents/graph/trading_graph.py)。它不直接做分析，而是负责把下列部件装配成一个可调用的 LangGraph：

| 组件 | 来源 | 作用 |
| --- | --- | --- |
| 双模型客户端 | `create_llm_client()` | 同时创建 `quick_thinking_llm` 与 `deep_thinking_llm`，把快模型分配给分析师/辩论节点，把深模型分配给研究经理与风控裁决节点 |
| 持久化检查点 | `SqliteSaver` | 把图执行状态写入 `graph_checkpoints.db`，并开启 SQLite WAL 模式，减轻并发写锁问题 |
| 角色记忆 | `FinancialSituationMemory` | 为多头研究员、空头研究员、交易员、研究经理、风控经理提供复盘记忆检索 |
| 工具节点 | `ToolNode` | 为市场、新闻、基本面、宏观、资金面等角色预留 LangGraph 工具调用入口 |
| 数据采集器 | `DataCollector` | 统一预抓取行情、新闻、财务、资金流和指标，避免短线/中线两个周期重复拉数 |
| 路由器 | `ConditionalLogic` | 决定分析师是否继续调用工具、研究辩论是否继续、风控是否打回交易员 |
| 图构造器 | `GraphSetup` | 把所有角色节点、条件边和结束边编译成 `StateGraph(AgentState)` |
| 传播器 | `Propagator` | 生成初始状态、设置 recursion limit、封装 LangGraph 调用参数 |
| 反思与信号处理 | `Reflector`、`SignalProcessor` | 在主图外做复盘记忆更新，以及从长文本里提取 `BUY / SELL / HOLD` 关键词 |

这意味着 `TradingAgentsGraph` 的职责更接近“工作流装配器”而不是“某一个智能体”。项目把提示词、数据工具、状态结构和条件路由全部外挂成独立模块，便于后续替换模型、裁剪角色或改写执行链。

### 2.2 角色分层：不是单链条，而是“四层协作 + 两级裁决”

`tradingagents/agents/` 里的角色可以分成六组，且每组消费的输入和写回的状态字段都不同。

| 层级 | 角色 | 主要输入 | 主要输出 |
| --- | --- | --- | --- |
| 基础分析层 | `market`、`social`、`news`、`fundamentals`、`macro`、`smart_money` | 预抓取数据、周期上下文、用户关注点 | `market_report`、`sentiment_report`、`news_report`、`fundamentals_report`、`macro_report`、`smart_money_report`，以及 `analyst_traces` |
| 博弈归纳层 | `Game Theory Manager` | `smart_money_report` + `sentiment_report` | `game_theory_report`、`game_theory_signals` |
| 投研辩论层 | `Bull Researcher`、`Bear Researcher`、`Research Manager` | 基础分析报告、博弈信号、历史记忆、claim 状态 | `investment_debate_state`、`investment_plan` |
| 交易执行层 | `Trader` | `investment_plan`、用户持仓/风险约束、风控反馈、历史记忆 | `trader_investment_plan` |
| 风控辩论层 | `Aggressive Analyst`、`Conservative Analyst`、`Neutral Analyst` | 交易员方案、各类研究报告、风险 claim 状态 | `risk_debate_state` |
| 最终裁决层 | `Risk Judge` | 风控辩论历史、交易员方案、用户上下文、历史记忆 | `risk_feedback_state`、`final_trade_decision` |

这里有两个很重要的设计点：

1. 基础分析层是“并行采样”，目的是从不同维度同时产出证据。
2. 后三层不是简单总结，而是把“研究结论 -> 可执行方案 -> 风险审查”拆成了三个不同视角的裁决过程。

因此，这套系统并不是六个 analyst 轮流写报告后直接结束，而是把“研究结论是否成立”和“即使结论成立，方案是否能执行”拆成两道独立关卡。

还要注意“角色存在”和“角色被后续主链直接消费”不是一回事。当前实现里，`Bull Researcher`、`Bear Researcher`、`Trader` 和 `Risk Judge` 直接读取的核心输入仍然是 `market_report`、`sentiment_report`、`news_report` 与 `fundamentals_report`；`smart_money_report` 主要先被 `Game Theory Manager` 抽象成 `game_theory_signals` 再进入研究经理；`macro_report` 则更多作为独立报告和最终日志的一部分保留下来，而不是当前收口节点的主输入。这说明仓库已经为更多分析维度预留了角色，但并非所有维度都同等深入地接进了最终裁决链。

### 2.3 共享状态：`AgentState` 是整条链的唯一事实来源

所有节点都围绕 [`tradingagents/agents/utils/agent_states.py`](../tradingagents/agents/utils/agent_states.py) 里的 `AgentState` 读写。它继承自 LangGraph 的 `MessagesState`，但在通用消息之外又加入了大量业务字段。可以把它理解成“本次分析任务的全量上下文 + 中间产物容器”。

| 状态分组 | 代表字段 | 含义 |
| --- | --- | --- |
| 标的与市场上下文 | `company_of_interest`、`trade_date`、`instrument_context`、`market_context` | 标准化股票代码、市场归属、交易时段、分析模式、数据截止时间 |
| 用户上下文 | `user_context`、`user_intent`、`workflow_context` | 调用方显式传入并标准化后的仓位/风险约束、自然语言意图解析结果、请求来源和已选 analyst |
| 分析报告 | `market_report`、`sentiment_report`、`news_report`、`fundamentals_report`、`macro_report`、`smart_money_report` | 六个 analyst 的原始结论文本 |
| 中间裁决 | `game_theory_report`、`game_theory_signals`、`investment_plan`、`trader_investment_plan` | 从情绪/主力博弈到研究经理计划，再到交易员可执行方案 |
| 辩论状态 | `investment_debate_state`、`risk_debate_state`、`risk_feedback_state` | 记录 claim、轮次、焦点问题、是否需要打回重写 |
| 结果与痕迹 | `final_trade_decision`、`analyst_traces`、`short_term_result`、`medium_term_result`、`metadata` | 最终决策、各 analyst 的结构化摘要、双周期输出和运行时附加信息 |

`Propagator.create_initial_state()` 在图启动前就会把这些字段填好默认值。它先做三件事：

1. 用 `infer_instrument_context()` 推断标的是 A 股还是美股，并补全交易所、币种和资产类型。
2. 用 `build_market_context()` 推断当前是盘前、盘中、盘后、历史回看还是未来日期，从而决定 `analysis_mode` 与 `data_as_of`。
3. 若调用方显式传入 `user_context`，就用 `normalize_user_context()` 清洗用户仓位、成本、止损比例等输入，再把“标的摘要 + 市场摘要 + 用户摘要”塞进 `messages` 作为所有角色共享的起始提示；若额外传入 `user_intent`，则原样保存在 `state["user_intent"]`。

这种做法有两个直接效果：

- 角色节点不需要自己重复判断“这是 A 股还是美股”“今天是盘前还是盘后”。
- 单周期入口 `propagate()` 可以把显式传入的用户约束直接送进交易员和风控提示词；但当前双周期入口 `propagate_async()` 只传了 `user_intent=user_intent`，没有同步传 `user_context=user_intent["user_context"]`，所以解析出的仓位/风险字段默认仍停留在 `user_intent` 中，不会自动进入 `build_agent_context_view()` 读取的 `state["user_context"]`。

还有一个容易忽略的细节：`analyst_traces` 在状态类型上声明为 `Annotated[List[TraceItem], operator.add]`。这表示多个 analyst 并行执行时会按追加方式合并，不会相互覆盖。它本质上是给最终报告和调试链路准备的“轻量化结构化摘要”。

### 2.4 Prompt 不是纯文案，而是控制平面的一部分

`tradingagents/prompts/catalog.py` 会根据配置里的 `prompt_language` 或 `prompt_language_by_provider`，在 [`tradingagents/prompts/zh.py`](../tradingagents/prompts/zh.py) 与 [`tradingagents/prompts/en.py`](../tradingagents/prompts/en.py) 之间切换。当前中文模板不是简单翻译，而是承担了三层职责：

1. 约束角色行为：例如市场分析师必须覆盖趋势、动量、波动和量价，不允许“堆指标”。
2. 注入周期视角：`build_horizon_context()` 会把短线/中线、关注点和具体问题拼进每个角色的系统提示。
3. 输出机读块：多个节点在正文结尾追加 HTML 注释，供图路由和状态归档解析。

当前代码实际依赖的机读块如下：

| 机读标记 | 生产节点 | 作用 |
| --- | --- | --- |
| `<!-- VERDICT: {...} -->` | 各 analyst、研究经理、交易员、风控裁决 | 提取方向性摘要，写入 `analyst_traces` 或供信号处理使用 |
| `<!-- GAME_THEORY: {...} -->` | `Game Theory Manager` | 产出结构化博弈信号，供研究经理收口 |
| `<!-- DEBATE_STATE: {...} -->` | 多头/空头研究员 | 更新投资辩论里的 claim、焦点 claim、轮次摘要和下轮目标 |
| `<!-- RISK_STATE: {...} -->` | 激进/保守/中性风控分析师 | 更新风险辩论里的 claim 和下一轮聚焦问题 |
| `<!-- RISK_JUDGE: {...} -->` | `Risk Judge` | 决定 `pass / revise / reject`，并给出硬约束、执行前提和降风险触发器 |

这套设计很关键。项目并没有使用单独的 JSON API 在节点之间传输中间裁决，而是让 LLM 生成“面向人阅读的正文 + 面向机器解析的 HTML 注释”。这样既保留了研报可读性，又让 LangGraph 能继续做条件路由。

### 2.5 图结构与条件路由：LangGraph 负责编排，状态字段决定去向

[`tradingagents/graph/setup.py`](../tradingagents/graph/setup.py) 明确了整张图的骨架。主路径可以概括为：

```text
START
  -> 六类 Analyst 并行
  -> Game Theory Manager
  -> Bull Researcher <-> Bear Researcher
  -> Research Manager
  -> Trader
  -> Aggressive -> Conservative -> Neutral 风控辩论
  -> Risk Judge
  -> END 或回退到 Trader
```

具体路由规则由 [`tradingagents/graph/conditional_logic.py`](../tradingagents/graph/conditional_logic.py) 控制：

- `should_continue_market/social/news/fundamentals/macro/smart_money`
  - 检查 `messages[-1].tool_calls` 是否存在。
  - 若存在则进入对应 `tools_*` 节点，再回到 analyst。
  - 若不存在则直接进入 `... Analyst Done`。
- `should_continue_debate`
  - 若 `investment_debate_state.count` 已达到 `2 * max_debate_rounds`，交给 `Research Manager` 收口。
  - 否则在 Bull 和 Bear 之间轮换。
- `should_continue_risk_analysis`
  - 若 `risk_debate_state.count` 已达到 `3 * max_risk_discuss_rounds`，交给 `Risk Judge`。
  - 否则在激进、保守、中性三方之间轮换。
- `should_revise_after_risk_judge`
  - 若 `risk_feedback_state.revision_required` 为真，且 `retry_count` 未超过 `max_retries`，则打回 `Trader` 重写方案。
  - 否则结束图执行。

这里还有一个实现层面的细节值得单独指出：图里虽然保留了 `tools_market`、`tools_news` 等 `ToolNode`，但按当前代码形态，这条 analyst -> `tools_*` -> analyst 的循环基本不会被走到。原因更接近“状态没有接上”，而不只是“调用频率低”：

1. 当前 analyst 节点返回的主要是 `market_report`、`news_report`、`analyst_traces` 之类的业务字段，并不会把 LLM 返回值写回 `state["messages"]`。
2. `ConditionalLogic.should_continue_*()` 判断是否进入 `ToolNode` 时，只检查 `state["messages"][-1].tool_calls`。
3. 由于初始 `messages` 只是 `Propagator` 写入的一条人类提示，且 analyst 节点不会覆盖这条消息，所以条件判断几乎总会落到 `"done"`。
4. `DataCollector` 缓存优先和提示词里的“不要继续请求工具”约束，只是进一步强化了这条保留分支不会成为主路径。

所以，当前版本的 `ToolNode` 更像是为旧式工具调用 prompt 或未来重接 `messages` 流保留的图结构，而不是现状下可正常触发的高频执行路径。

### 2.6 数据获取与双周期执行：短线、中线共用一套数据池

单次分析有两种入口：

- `propagate()`：单周期执行，输入 `company_name + trade_date + user_context`，返回完整最终状态和提炼后的交易信号。
- `propagate_async()`：双周期执行，输入 `company_name + trade_date + query`，并行跑 `short` 与 `medium` 两个 horizon。

双周期路径更能代表当前产品形态。其关键流程如下：

1. 若用户给了自然语言 `query`，先调用 `parse_intent()` 提取 `ticker`、`horizons`、`focus_areas`、`specific_questions` 与 `user_context`。
2. `DataCollector.collect()` 先把所需行情、新闻、全球新闻、板块资金、个股资金、龙虎榜、财务报表和技术指标并行抓完，缓存在内存里。当前 `propagate_async()` 调它时没有传 `horizons`，因此默认走的是全量抓取，而不是 `horizons == ['short']` 时才会触发的 14 天 `short_only` 优化。
3. 对每个 horizon，`Propagator.create_initial_state()` 会构建一份独立状态，并把解析结果存进 `user_intent`；但当前实现没有把 `user_intent["user_context"]` 同步到 `state["user_context"]`，所以双周期路径里的交易员/风控提示词默认看不到这些解析出的持仓字段。
4. 各 analyst 会读取同一份缓存池，但 `DataCollector.get_window()` 目前主要只是给缓存副本补上 `_data_window` 和 `_horizon` 元数据，并不会按 horizon 真正裁切成两份不同的数据集。也就是说，双周期的差异主要体现在 prompt 侧重点、个别 analyst 的直连兜底抓取逻辑，以及 `analyst_traces` 里的窗口标签，而不是“同一缓存被切成两段不同时间窗”。
5. 图执行完成后，`_build_horizon_result()` 从完整状态里提取双周期摘要，最终合成为 `short_term` 和 `medium_term` 两个结果块。
6. 运行结束后，缓存通过 `DataCollector.evict()` 释放，避免长时间堆积。

最小调用示例如下：

```python
from tradingagents.graph.trading_graph import TradingAgentsGraph

graph = TradingAgentsGraph(debug=False)
result = await graph.propagate_async(
    company_name="600519",
    trade_date="2026-03-18",
    query="我半仓持有 600519，成本 1500，偏保守，想看短线和中线怎么操作。",
)

print(result["short_term"]["final_trade_decision"])
print(result["medium_term"]["final_trade_decision"])
```

这个例子体现了第 2 章最核心的事实：系统不是只返回一个大段文本，而是先把用户意图解析成结构化结果并挂到 `user_intent`，再按不同时间维度各跑一遍同一张图。需要注意的是，当前双周期实现还没有把解析出的 `user_context` 自动桥接进交易员和风控节点读取的 `state["user_context"]`。

### 2.7 从请求进入到报告产出的主执行链

把上面的模块拼起来，一次标准分析请求会经过下面这条链：

1. 请求进入后端，由服务层把股票代码、交易日、自然语言问题和用户约束传给 `TradingAgentsGraph`。
2. `parse_intent()` 尝试从自然语言里抽取周期、关注点、持仓、成本、止损容忍度等信息；如果 LLM 解析失败，还会用正则做兜底。
3. `Propagator` 构造 `AgentState`，提前写入标的上下文、市场上下文、默认辩论状态和 `messages` 起始提示。
4. 六类 analyst 并行生成基础报告，并把各自结论压缩成 `analyst_traces`。
5. `Game Theory Manager` 只消费情绪和主力资金两个维度，生成“主力 vs 散户”的博弈判断。
6. `Bull Researcher` 与 `Bear Researcher` 围绕 `claim` 交替辩论，`Research Manager` 在达到轮次上限后收口并输出 `investment_plan`。
7. `Trader` 把研究计划翻译成可执行交易方案，并显式纳入风控反馈和历史记忆；只有在单周期入口或调用方显式填充 `state["user_context"]` 时，用户仓位、成本和风险约束才会直接出现在它的上下文摘要里。
8. 激进、保守、中性三位风控分析师围绕执行风险继续辩论，更新 `risk_debate_state`。
9. `Risk Judge` 给出最终裁决，并写入 `risk_feedback_state`：
   - `pass`：直接结束。
   - `revise`：把硬约束和修订原因打回给 `Trader`，允许一次重写。
   - `reject`：结束并保留否决意见。
10. `TradingAgentsGraph` 把结果记录到 `eval_results/<ticker>/TradingAgentsStrategy_logs/`，并通过 `SignalProcessor` 从长文本中再提炼一次 `BUY / SELL / HOLD`。

用一张简化图表示，就是：

```text
自然语言/代码输入
  -> 意图解析与上下文标准化
  -> 数据预抓取与缓存
  -> 六类 Analyst 并行出报告
  -> 主力/情绪博弈归纳
  -> 多空研究辩论
  -> 研究经理收口
  -> 交易员生成执行方案
  -> 三方风控辩论
  -> 风控裁决
  -> 记录日志、输出最终报告与 BUY/SELL/HOLD
```

### 2.8 这一章可以带走的结论

理解第 2 章后，你应该能抓住这套引擎的三个本质：

1. 它的“多智能体”重点不在于角色数量，而在于把研究、执行和风控拆成可回退的多级裁决链。
2. 它的“共享状态”不是简单消息列表，而是带有 claim、周期、用户约束和结构化机读块的业务状态机。
3. 它的“图编排”并不神秘，核心就是 LangGraph 的并行 fan-out、条件路由和有限轮次辩论。

理解了这一点，再去看[第 3 章](#chapter-3)的后端 API 和任务落库，就能更清楚地知道：服务层到底是在调用一个“大模型接口”，还是在调度一条具备状态、回退和持久化能力的投研工作流。模型接入、提示词语言切换和运行配置则放到[第 5 章](#chapter-5)展开。

<a id="chapter-3"></a>
## 第3章：后端 API、服务层与数据持久化

> 待补充。

<a id="chapter-4"></a>
## 第4章：前端页面、状态管理与交互链路

> 待补充。

<a id="chapter-5"></a>
## 第5章：数据源、模型接入、配置与部署方式

> 待补充。

<a id="chapter-6"></a>
## 第6章：全文审校与整合

> 待补充。
