# agent-lab

一个 Java 后端工程师的 **24 周 AI 智能体开发**自学仓库：逐日讲义 + 可运行代码 + 两个作品集项目。

工作日每天 1 小时、周六 2 小时、周日休息。约 168 小时有效投入，目标是做出能被别人使用、能被量化评估的 Agent 项目，用于内部转岗或外部应聘 AI 应用工程方向。

## 怎么用这个仓库

每天只做一件事：打开当周讲义，找到今天那一节。

```bash
# W01 周一开始，按讲义执行；讲义里有每天的具体命令和代码
open docs/lessons/W01-Python语法-Java对照.md
```

三种结构，别按同一套读法硬套：

- **普通周（周一到周五）**每天七段：**本质 → Java 视角 → 核心讲解 → 代码 → 今天的动作 → 会踩的坑 → 自测**。
- **周六**是 120 分钟深度块，只有分段任务清单与休息节点，没有七段。
- **清账周（W04/W08/W12/W16/W20/W24）**是另一套：诊断表 → 补练讲义 → 阶段验收 → 每日安排 → 范围判定指引。

收尾前必须 `git commit`，没有提交就视为当天未完成。

三份文档分工：

| 位置 | 作用 | 什么时候看 |
|---|---|---|
| `docs/plan/` | 任务表：每周每天做什么 | 周一早上确定本周范围 |
| `docs/lessons/` | 讲义：概念、代码、坑、自测 | 每天晚上的 1 小时 |
| `learning-log.md` | 你的记录：目标、卡点、明天第一步 | 每天开场 5 分钟与收尾 10 分钟 |

## 目录结构

```
agent-lab/
├── docs/
│   ├── plan/          # 24 周任务表（按阶段分 6 个文件 + 总览）
│   ├── lessons/       # 逐日讲义，一天一节
│   └── notes/         # 我自己写的笔记与复盘（W04 起）
├── src/
│   ├── llm/           # 客户端、流式、token、历史裁剪
│   ├── prompts/       # 提示词模板与加载器
│   ├── tools/         # 工具实现与 schema
│   ├── agent/         # 工具调用循环、上下文构建、图
│   └── retrieval/     # 解析、切分、索引、检索、重排
├── mcp_servers/       # W18：自建 MCP server
├── evals/             # W21：评测集、judge、回归脚本
├── experiments/       # 对比实验的原始数据与结论（技术文素材）
├── projects/
│   ├── p1-assistant/  # W11：单 Agent 工具型应用
│   └── p2-workflow/   # W17：多智能体 + MCP 工作流
├── scripts/           # CLI 入口与一次性脚本
├── tests/
└── learning-log.md
```

## 24 周索引

| 周 | 讲义 | 主题 | 产出 |
|---|---|---|---|
| W01 | [Python 语法（Java 对照）](docs/lessons/W01-Python语法-Java对照.md) | 环境与语法速通 | 环境 + 对照表 |
| W02 | [Python 工程能力](docs/lessons/W02-Python工程能力.md) | httpx / pydantic / asyncio / 配置 / pytest | 第一个真实模型调用 |
| W03 | [LLM API 的本质](docs/lessons/W03-LLM-API的本质.md) | 消息结构 / token / 流式 / 裁剪 | `llm_lab.py` |
| W04 | [清账周](docs/lessons/W04-清账周-阶段0收口.md) | 补欠账 + 阶段自检 | `src/llm/` 定型 |
| W05 | [Prompt 与结构化输出](docs/lessons/W05-Prompt与结构化输出.md) | 五要素 / few-shot / 输出契约 | 可校验的输出 |
| W06 | [Function Calling](docs/lessons/W06-Function-Calling.md) | 工具定义 / 循环 / 写操作安全 | `agent_loop.py` |
| W07 | [上下文工程](docs/lessons/W07-上下文工程.md) | 预算分配 / 压缩 / 缓存 | 上下文装配器 |
| W08 | [清账周](docs/lessons/W08-清账周-阶段1.md) | 阶段 1 诊断与验收 | 工具循环可重放 |
| W09 | [ReAct 与 Agent 循环](docs/lessons/W09-ReAct与Agent循环.md) | 手写 agent 内核 | 手写循环 + 校验 |
| W10 | [LangGraph 状态图](docs/lessons/W10-LangGraph状态图.md) | 图 / 条件边 / 断点续跑 | 带回环的状态图 |
| W11 | [项目一：端到端最小闭环](docs/lessons/W11-项目一-端到端最小闭环.md) | PRD / 工具集 / 打通 | 能跑的项目一 |
| W12 | [清账周](docs/lessons/W12-清账周-项目一收口.md) | 阶段 2 验收 | 可试用版本 |
| W13 | [RAG 地基](docs/lessons/W13-RAG地基-解析切分向量.md) | 解析 / 切分 / 向量库 | 最小可检索 |
| W14 | [检索质量与评测](docs/lessons/W14-检索质量与评测.md) | 混合检索 / rerank / recall | 检索对比表 |
| W15 | [RAG 工程化与记忆](docs/lessons/W15-RAG工程化与记忆.md) | 增量索引 / 记忆分层 | 项目一 v2 |
| W16 | [清账周](docs/lessons/W16-清账周-项目二选题.md) | 阶段 3 验收 + 选题 | 项目二 PRD |
| W17 | [多智能体编排](docs/lessons/W17-多智能体编排.md) | supervisor / 拆分判据 | 项目二骨架 |
| W18 | [MCP](docs/lessons/W18-MCP.md) | client / server / 治理 | 自建 MCP server |
| W19 | [长任务与可靠性](docs/lessons/W19-长任务与可靠性.md) | 断点续跑 / 幂等 / 执行边界 | 可靠运行层 |
| W20 | [清账周](docs/lessons/W20-清账周-阶段自检.md) | 阶段 4 验收 + 框架选型表 | 项目二可演示 |
| W21 | [评估与可观测](docs/lessons/W21-评估与可观测.md) | 评测集 / LLM-as-judge / trace | 回归命令 |
| W22 | [部署、成本与安全](docs/lessons/W22-部署成本与安全.md) | FastAPI / 容器 / 成本 / 攻防 | **项目一公网 URL** |
| W23 | [作品集与 Java 差异化](docs/lessons/W23-作品集与Java差异化.md) | README / 技术文 / Spring AI | 技术文 + 作品集 |
| W24 | [冲刺与总复盘](docs/lessons/W24-冲刺与总复盘.md) | 自评 / 系统设计 / 投递 | 材料已发出 |

## 硬性规矩

1. 周日不打开这个仓库。
2. 密钥只进 `.env`，永不入库；提交过一次就必须吊销。
3. 每天以 `git commit` 结尾。
4. 每第四个周六做阶段自检，未达标项写进下一阶段任务，不追加时间。
5. 日志与实验里不记录用户隐私与内部数据原文。

## 环境

Python 3.12（用 [uv](https://docs.astral.sh/uv/) 管理，W01 周一装）。IDE 解释器必须指向项目内 `.venv`。

```bash
uv sync                 # 装依赖
uv run pytest -q        # 测试
uv run pyright          # 静态检查
```
