# agent-lab

一个 Java 后端工程师的 **24 周 AI 智能体开发**自学仓库：逐日讲义 + 可运行代码 + 两个作品集项目。

工作日每天 1 小时、周六 2 小时、周日休息。约 168 小时有效投入，目标是做出能被别人使用、能被量化评估的 Agent 项目，用于内部转岗或外部应聘 AI 应用工程方向。

## 怎么用这个仓库

每天只做一件事：打开当周讲义，找到今天那一节。

```bash
# W01 周一开始，按讲义执行；讲义里有每天的具体命令和代码
open docs/lessons/W01-Python语法-Java对照.md
```

每节讲义固定五段：**本质 → Java 视角 → 核心讲解 → 代码 → 今天的动作 → 会踩的坑 → 自测**。收尾前必须 `git commit`，没有提交就视为当天未完成。

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
| W05–W08 | 提示工程与工具调用 | Prompt / Function Calling / 上下文工程 | `agent_loop.py` |
| W09–W12 | Agent 核心与项目一 | ReAct / LangGraph / 项目一 | 可试用的项目一 |
| W13–W16 | RAG 与记忆 | 切分 / 混合检索 / rerank / 记忆分层 | 检索质量对比表 |
| W17–W20 | 多智能体与 MCP | supervisor / MCP server / 长任务可靠性 | 项目二 |
| W21–W24 | 评估、部署与出手 | 评测集 / trace / 公网部署 / 作品集 | URL + 技术文 + 投递材料 |

（W05 起的讲义文件按周陆续落在 `docs/lessons/`。）

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
