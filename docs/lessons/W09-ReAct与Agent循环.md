# W09 · Agent 循环原理（手写 ReAct）

> 本周在计划中的位置：阶段 2 第一周。上一阶段你已经手写过 `agent_loop.py`（模型↔工具循环），本周不是重做它，而是回答两个更底层的问题——"什么才算 agent"和"这个循环怎样才能不被自己绊倒"。下周（W10）才允许碰框架，本周一切手写。

## 本周导航

| 日 | 主题 | 今天的产出物 |
|---|---|---|
| 周一 | workflow 与 agent 的分界 | 一页自己话写的判据笔记 `notes/w09_workflow_vs_agent.md` |
| 周二 | 文本协议版 ReAct | `experiments/w09/react_text.py` + 解析失败清单 |
| 周三 | 原生 tool calling 版 ReAct | `experiments/w09/react_native.py` + 两版对比表 |
| 周四 | 完成判定与终止 | `experiments/w09/validator.py`，校验失败回环 |
| 周五 | JSONL 落盘追踪 | `experiments/w09/tracer.py`，一次运行一个 `.jsonl` |
| 周六 | 深度块：断点重放 | replay 脚本 + 一次真实故障的定位记录 |
| 周日 | 休息，不许补课 | — |

---

## 周一 · workflow 与 agent 的分界

**本质**：Anthropic《Building effective agents》的核心论点不是"agent 好"，而是**大多数场景你该用 workflow，但你必须知道自己在用哪个**。workflow 的控制流（先做什么、后做什么、走哪个分支）写死在你的代码里；agent 的控制流由模型在运行时决定——它自己决定下一步调哪个工具、还要不要继续。这个区别决定了你的成本、延迟、可评估性和故障模式，全部不一样。

**Java 视角**：workflow 像用 Flowable/Activiti 画的 BPMN 流程——节点和转移条件全是代码定义的，引擎只是执行者。agent 更像把决策权交给一个外包操作员：你只给他工具和目标，路径他自己编。审计和兜底思路完全不同。

**核心讲解**：

论文里 workflow 与 agent 的定义值得原样记住：workflow = "LLM 和工具通过预定义的代码路径编排"；agent = "LLM 动态决定如何完成任务，自主使用工具并管理自己的过程"。复述结论没用，你要的是**判据**。给你三条可操作的判据，任一条命中即为 agent 成分：

1. **循环出口由谁决定**：模型输出里出现"还要不要继续"的信号（如发起新 tool_call、声明完成）并且这个信号直接控制循环，是 agent；出口是你写的 `if`（比如固定"先抽取→再检索→再生成"三步），是 workflow。
2. **步数是否随输入变化**：同一系统对不同输入走 1 步或 8 步，是 agent；步数恒定或只按你枚举的条件分支走，是 workflow。
3. **分支可枚举性**：你能不能在设计评审时把"所有可能执行路径"画在白板上。能画全→workflow；画不全且你承认画不全→agent。

为什么多数"Agent 项目"其实是 workflow：很多产品把三段固定 prompt 串起来（分类→检索→总结）对外叫 agent，因为营销需要。反过来也有一种错误是把能枚举的场景硬做成自主循环，白白多付 5-10 倍 token 和延迟，还引入了死循环风险。**正确的叙事是：你知道边界在哪，并且按场景选择**。这句话写进你今天的笔记，也是阶段 2 验收第一条的答题方向。

Agent 的代价要写实：延迟不可预算（步数不定）、成本不可预算（同上）、评估难（同输入不同轨迹）、故障模式新增（循环、工具误用、幻觉级联）。Agent 的收益也写实：你不需要为没见过的问题预先写分支。

**代码**：今天不写代码，用代码思维整理判据。下面这个函数当作你笔记的骨架：

```python
from enum import Enum

class Style(Enum):
    WORKFLOW = "workflow"   # 控制流全部预定义
    AGENT = "agent"         # 控制流由模型输出决定
    HYBRID = "hybrid"       # 外层 workflow，内层局部 agent

def classify(paths_enumerable: bool, exit_by_model: bool, steps_vary: bool) -> Style:
    """不是运行时函数，是你做选型时的决策表。"""
    votes = [exit_by_model, steps_vary, not paths_enumerable]
    if sum(votes) == 0: return Style.WORKFLOW
    if sum(votes) == 3: return Style.AGENT
    return Style.HYBRID  # 大多数真实系统落在这，要点是标出 agent 成分在哪一层
```

**今天的动作**：
- [ ] 午休读完《Building effective agents》全文（20-30 分钟），只标记不懂的段落，不查第三方解读。
- [ ] 晚上 45 分钟：写 `notes/w09_workflow_vs_agent.md`，必须含：三条判据的自己的话版本 + 用判据重新审视你 W06 的 `agent_loop.py`（它是 agent 还是 hybrid？为什么）+ 你司现有某个"AI 功能"的归类。
- [ ] 自问：如果面试官问"这场景为什么用/不用 agent"，这份笔记能否直接支撑答案。能→今天的产出达标。
- [ ] commit + 三行日志。

**会踩的坑**：
1. **把"用了 LLM"当成"是 agent"**。判据 1 能一票否决：LLM 只在固定步骤里做一次生成，是 workflow 节点，不是 agent。
2. **笔记写成论文摘要**。论文是输入，判据是输出；通篇复述"agents excel at open-ended problems"却没有任何一条你能用的判断规则，就是没做事。
3. **立刻想做"那框架怎么选"**。今天禁止，W10 只学 LangGraph 一个是刻意安排。

**自测**：
1. 一个系统：固定调 2 次 LLM（改写查询→生成答案），但检索词由模型自由输出。它是 workflow 还是 agent？（答案：workflow；模型输出内容自由≠控制流自由。）
2. 为什么 agent 的 P99 延迟天然难承诺？用"步数随输入变化"推出来说。
3. 给出一个你工作里明确该用 workflow 的场景和一个该用 agent 的场景，各一句理由。

---

## 周二 · 文本协议版 ReAct：亲手体验脆弱

**本质**：ReAct（Reason + Act）论文的形态是让模型在**纯文本输出**里交替产生 Thought（推理）/ Action（选工具）/ Action Input（工具入参），你的代码解析这些字段、执行工具、把 Observation 拼回下一轮 prompt。在没有原生 tool calling 的年代（2022）这是唯一方案；今天手写它是为了**建立对协议脆弱性的体感**——这是你周三理解"为什么 tool calling 更稳"的对照组。

**Java 视角**：等于用正则去 parse 一个没有 schema 契约、由对方（模型）即兴生成的私有文本协议。Java 里你会用 Jackson + JSON Schema 校验；这里契约只存在于提示词的"口头约定"中，实现端全靠正则兜——所有口头约定都会在你没想到的输入上违约。

**核心讲解**：

解析循环的协议长这样（固定进 system prompt）：

```
Question: ...
Thought: ...   Action: tool_name   Action Input: {"arg": "value"}
Observation: ...        （以上三行一组，可重复若干轮）
Thought: I know the answer now.   Final Answer: ...
```

字符串解析会在这些地方断裂，每一条都是真实高频表现：

1. **全角/中英文混用**：模型输出 `Action：搜索`（全角冒号+中文工具名），你的 `r"^Action:\s*(\w+)"` 直接失配。中文场景出现率显著高于英文。
2. **Action Input 被代码块包裹**：模型输出 ` ```json {...} ``` `，逐行解析把围栏当内容，`json.loads` 报错。
3. **嵌套 JSON 里的换行**：协议用"行"做字段边界，但 Action Input 的 JSON 值里一旦有格式化换行，解析器只取到第一行 `{`。
4. **一条回复里两个 Action**：模型合并思考，你的循环只取第一个，第二个静默丢失——最难查，因为"看起来跑对了"。
5. **Final Answer 的大小写/加粗变体**：`**Final Answer:**`、`Final answer:` 都要匹配，漏一个就永远不终止直到撞最大步数。
6. **模型自己编造 Observation**：它见过太多"Observation: 结果"的模式，会顺手替你生成假的工具返回。对策是 Observation 必须用显式分隔符包裹并在 prompt 里声明"此块内容来自外部，非你所写"，但这是缓解不是根治。

**今天的动作不是把解析做多优雅，而是把失败计数打开**：每次解析失败记一行（原因分类 + 原始文本），跑 10 个问题后你会拿到一张脆弱性账单——这就是周三对比表的左半边。

**代码**：`experiments/w09/react_text.py` 核心解析器（Python 3.12）：

```python
import json, re
from dataclasses import dataclass
from typing import Any

@dataclass
class ParsedStep:
    kind: str            # "action" | "final" | "invalid"
    thought: str = ""
    tool: str | None = None
    args: dict[str, Any] | None = None
    final: str | None = None
    error: str | None = None

# 全部正则带 IGNORECASE 且冒号写成 [:：]（全角冒号是中文场景实测高频坑）
FINAL_RE   = re.compile(r"final\s+answer\s*[:：]\s*(.+)", re.I | re.S)
ACTION_RE  = re.compile(r"^\s*action\s*[:：]\s*([A-Za-z0-9_\u4e00-\u9fff]+)", re.I | re.M)
INPUT_RE   = re.compile(r"action\s+input\s*[:：]\s*", re.I)
THOUGHT_RE = re.compile(r"thought\s*[:：]\s*(.+?)(?=\n\s*(?:action|final)|\Z)", re.I | re.S)

def parse_react_turn(text: str) -> ParsedStep:
    """解析模型的一轮输出。宁可返回 invalid 也不要猜。"""
    if len(ACTION_RE.findall(text)) > 1:
        return ParsedStep("invalid", error="multiple_actions")  # 静默取第一个是事故源
    m = FINAL_RE.search(text)
    if m and not ACTION_RE.search(text):
        return ParsedStep("final", final=m.group(1).strip())
    am, im = ACTION_RE.search(text), INPUT_RE.search(text)
    if not am:
        return ParsedStep("invalid", error="no_action_line")
    if not im:
        return ParsedStep("invalid", error="missing_action_input", tool=am.group(1))
    raw = re.sub(r"^```(?:json)?\s*|\s*```$", "", text[im.end():].strip(), flags=re.S)  # 剥 markdown 围栏
    try:
        # JSON 可能跨行且后面还跟着别的东西：raw_decode 只解析前缀，loads 会全量报错
        args, _ = json.JSONDecoder().raw_decode(raw)
    except json.JSONDecodeError as e:
        return ParsedStep("invalid", error=f"bad_json:{e.msg}", tool=am.group(1))
    th = THOUGHT_RE.search(text)
    return ParsedStep("action", tool=am.group(1), thought=th.group(1).strip() if th else "",
                      args=args if isinstance(args, dict) else {"query": args})
```

循环骨架（今天先跑通一个工具就够，比如 `search` 用 W06 的假实现）：`messages` 里不断追加模型输出、`Observation: <...>` 字符串，`while step_idx < MAX_STEPS`，解析 invalid 时**把失败原因原文回喂**（"你的输出不符合协议，请重新输出，注意 Action Input 必须是单行 JSON"）。**硬性习惯：`MAX_STEPS` 上限（建议 8）和单次运行的 token 累计上限（超了就 break 并标记 `terminated_by="budget"`），没有上限的循环等于给账单开了无护栏口子。**

**今天的动作**：
- [ ] 写 system prompt 模板文件放 `prompts/react_text.md`，协议原文 + 一版完整示例（示例顺序：Thought→Action→Action Input→Observation，含 Final Answer 一例）。
- [ ] 实现 `parse_react_turn` + 循环，工具先用 1 个假 `search`。
- [ ] 跑 10 个问题（5 个单跳 + 3 个多跳 + 2 个刁钻格式），统计解析失败次数与分类，写进 `experiments/w09/parse_failures.md`。
- [ ] 验证：故意让一个失败样例重跑 3 次，观察是否同型复发（是→协议的随机违约，不是→你的 prompt 有歧义）。
- [ ] commit + 日志。

**会踩的坑**：
1. **用 `json.loads` 而不是 `raw_decode`**：Action Input 后面常跟着多余文本，`loads` 全量报错，`raw_decode` 只解析前缀 JSON。
2. **解析失败就整轮重试**：白白消耗 step；正确做法是把错误信息回喂一次、只重试该轮输出。
3. **忘了转义 Observation**：工具返回里含 `Final Answer:` 字样的日志文本时，解析器把工具输出当模型结论提前终止。对策：解析只针对模型消息，不扫全量上下文。

**自测**：
1. 六个脆弱性里哪三个是"静默错误"（不报错但结果错）？（多 Action 取首、Observation 截断、假 Observation。）静默为什么比崩溃危险？
2. 为什么解析器要"宁可 invalid 也不猜"？猜的代价在哪个环节暴露？
3. 你的失败账单上哪一类占比最高？周三后能用数据回答。

---

## 周三 · 原生 tool calling 版 ReAct

**本质**：同一个任务循环，把"文本协议 + 正则解析"换成模型 API 的原生 tool calling：工具定义以 JSON Schema 随请求发出，模型返回结构化的 `tool_calls` 字段，`arguments` 是合法 JSON 字符串。执行权、循环权仍然全在你手里——和 W06 没变，变的是**协议承载方式**。直接复用你的 `agent_loop.py`，今天做的是"改造 + 对照实验"。

**Java 视角**：从"解析对方发的自由文本"升级到"对方按你给的接口定义（IDL）生成调用请求"，序列化层由 provider 的 constrained decoding 兜底。类比 Dubbo/Feign：方法签名是契约，你不再手写字符串拆包。

**核心讲解**：

为什么更稳，机制层面三个原因（背结论没用，记这三条）：

1. **解码时约束**：主流 provider 对 `tool_calls.arguments` 做 schema 约束下的生成（constrained decoding / 微调对齐），"输出非法 JSON"这类错误在源头上被压掉。文本协议里模型的每个 token 都在自由发挥。
2. **协议与内容分离**：`content`（自然语言）和 `tool_calls`（结构化意图）是两个字段，不会互相污染。周二"工具输出里含 Final Answer 字样被误判"这类串扰在原生协议下不存在——`finish_reason == "tool_calls"` 就是终止判断的全部依据，无需扫文本。
3. **关联 id 由 provider 生成**：每个 tool_call 带唯一 `id`，你的执行结果以 `ToolMessage(tool_call_id=id)` 回填。多工具并行时靠 id 配对，文本协议里只能靠顺序猜——顺序一旦错位，后面全错且很难发现。

对照实验设计（这是今天真正的产出）：固定同一 10 个问题、`temperature=0`，两版各跑一遍，记四个指标：**协议错误率**（解析失败或非法入参次数/总步数）、**任务通过率**、**平均步数**、**平均总 token**（input+output 分列）。预期结果与解释：文本协议错误率明显高（5-15%/步很常见）；token 上文本协议的 output 更长（Thought/Action 样板）且 input 随 scratchpad 滚雪球，原生版每步 input 也涨但增速低（tool 消息比手写 Observation 紧凑）；原生版有隐性成本——工具 schema 每请求都占 input token，工具多时不小。

**代码**：`experiments/w09/react_native.py`，核心是把 W06 循环改造成"带计数的可插桩版本"：

```python
from openai import OpenAI            # 1.x 风格，沿用 W06 起的约定
from pydantic import BaseModel, Field

client = OpenAI()  # key 在 .env，绝不硬编码

class SearchArgs(BaseModel):
    query: str = Field(min_length=1)

TOOLS = [{"type": "function", "function": {"name": "search",
          "description": "全文检索，返回最相关的若干条片段。",
          "parameters": SearchArgs.model_json_schema()}}]

def run_native(question: str, max_steps: int = 8) -> dict[str, object]:
    msgs: list[dict] = [{"role": "user", "content": question}]
    for step in range(1, max_steps + 1):                 # 硬上限，撞线即标记 terminated_by
        resp = client.chat.completions.create(model="你的主力模型", messages=msgs,
                                              tools=TOOLS, temperature=0)
        choice = resp.choices[0]
        msgs.append(choice.message.model_dump())          # assistant 原样回写，别手拼丢 tool_calls
        if choice.finish_reason != "tool_calls":
            return {"ok": True, "steps": step, "answer": choice.message.content,
                    "tokens": resp.usage.model_dump()}
        for tc in choice.message.tool_calls:
            try:
                # provider 只约束 JSON 语法层；语义合法性（空查询/超长/越权）靠 SearchArgs 兜
                args = SearchArgs.model_validate_json(tc.function.arguments)
                out = do_search(args.query)               # 你的假实现
            except Exception as e:
                out = {"error": f"{type(e).__name__}: {e}"}  # 失败也走 tool 消息回填，不 raise 出循环
            msgs.append({"role": "tool", "tool_call_id": tc.id,
                         "content": str(out)[:2000]})      # 截断保上下文预算（W07 的账在这还）
    return {"ok": False, "reason": "max_steps", "steps": max_steps}
```

**今天的动作**：
- [ ] 把 `agent_loop.py` 复制为 `react_native.py` 改造（别在原件上动，对照组要保真）。
- [ ] 加指标计数器：协议错误 / 步数 / token / 耗时，两版用同一套 `Metrics` dataclass。
- [ ] 跑对照实验：同 10 题 × 2 版，`temperature=0`，结果填 `experiments/w09/compare.md` 四指标表。
- [ ] 验证：表格下方写 3-5 行结论，其中必须有一条是"意料之外"的观察（没有就重跑换题）。
- [ ] commit + 日志。

**会踩的坑**：
1. **assistant 消息回写时丢了 `tool_calls` 字段**：把 `message.content` 手工重新拼，下一轮模型看不到自己调用过什么，重复调用。用 `message.model_dump()` 原样序列化。
2. **多个并行 tool_calls 只回填一个结果**：provider 校验 `tool_call_id` 配对，缺一个直接 400。for 循环里每个 tc 都要有对应 tool 消息。
3. **以为原生=不用校验入参**：语法层被约束了，语义层（空查询、超长查询、越权参数）没有。`model_validate` 一步不能省。

**自测**：
1. `finish_reason == "tool_calls"` 在你循环里承担什么角色？和周二扫文本找 "Final Answer" 的本质区别？
2. 为什么原生版的 token 优势可能随工具数量增多而缩水？（schema 进 input。）什么手段能对冲？（W07 的 prompt caching，固定工具前缀。）
3. 文本协议版在什么情况下仍然是正确选择？至少想两个：不支持 tool calling 的 endpoint、需要把"Thought"显式暴露给用户的场景……写进笔记。

---

## 周四 · 完成判定：模型说"做完了"不算数

**本质**：循环的**终止**和**完成**是两件事。终止是控制流层面（模型不再发 tool_call、或撞了步数上限）；完成是语义层面（输出真的满足任务要求）。模型自述"我做完了"不可信——不是它想骗你，而是（a）RLHF 训练偏好"给出一个答案"而非"承认没答案"，（b）它看不到你没定义的业务约束，（c）多轮工具结果里它可能引用错。所以需要一个**显式校验节点**：终止≠放行，放行只由校验决定。

**Java 视角**：就是接口返回 DTO 上的 Bean Validation。你不会信任 service 层"我保证字段非空"，而是在边界用 `@NotNull`/`@Valid` 硬校验，校验失败返回 400。这里同理：400 的处理方式是把校验错误回喂模型、再来一轮。

**核心讲解**：

校验分三层，按成本从低到高排，**能用下层绝不用上层**：

1. **格式校验（零成本）**：输出是合法 JSON / 含必需的 Markdown 段落 / 长度在区间内。正则和 `json.loads` 的事。
2. **契约校验（零成本，pydantic）**：定义 `FinalOutput` 模型，字段级约束（`min_length`、`Literal` 枚举、`field_validator`）。这是 Java 工程师的主场：把"什么算对"写成 schema。
3. **语义校验（花钱）**：来源引用是否真实存在于本轮工具返回中（这条可以**程序判定**，不用 LLM：抽出 answer 引用的行号/条目 id，对工具结果做子串/集合核对）、数字是否来自 Observation 而非脑补。真正做不了程序判定的才上"LLM as judge"，且 judge 也要有上限次数。

失败处理策略要有预算：校验失败→把**具体错误列表**回喂（"缺少字段 root_cause；引用的 L123 在检索结果中不存在"）→重新生成，**修复轮数上限 2**；超限后降级：输出最后一次结果 + `completed=False` + 失败原因列表，交给调用方决定。**绝不允许无限修复循环**——那是最隐蔽的烧钱模式。降级输出的 `terminated_by` 字段区分：`model_done_validated` / `repair_exhausted` / `max_steps` / `budget`，这直接对应阶段验收里"完成判定不能信模型自述"的证据链。

**代码**：`experiments/w09/validator.py`（接在 `react_native.py` 的返回路径上）：

```python
from pydantic import BaseModel, field_validator

class FinalOutput(BaseModel):
    root_cause: str
    evidence_lines: list[int]           # 引用的日志行号，必须来自工具返回
    suggestion: str

    @field_validator("evidence_lines")
    @classmethod
    def not_empty(cls, v: list[int]) -> list[int]:
        if not v:
            raise ValueError("evidence_lines 不能为空")
        return v

def validate_final(raw: str, observations: list[str]) -> tuple[FinalOutput | None, list[str]]:
    """返回 (结果, 人可读的错误列表)。错误列表将原样回喂模型——具体到字段，不许笼统。"""
    try:
        out = FinalOutput.model_validate_json(raw)
    except Exception as e:
        return None, [f"输出不是符合 schema 的 JSON: {e}"]
    known = {ln for obs in observations for ln in parse_line_ids(obs)}
    fake = [ln for ln in out.evidence_lines if ln not in known]
    if fake:
        return None, [f"引用了不存在的日志行: {fake}，可用行号: {sorted(known)[:20]}"]
    return out, []

# 主循环接法（伪代码）：finish_reason != "tool_calls" 后进入校验回环，
# repair < 2 轮：失败把 errs 作为 user 消息回喂再请求一次；
# 超限则返回降级结果并标 terminated_by="repair_exhausted"（绝不无限修复）。
```

**今天的动作**：
- [ ] 给你的 ReAct 任务定一个"完成契约"：用 pydantic 写出 `FinalOutput`，字段至少含结论、证据引用、下一步建议。
- [ ] 实现 `validate_final`，接入 `react_native.py` 的终止分支（模型不再调工具≠结束，校验过了才结束）。
- [ ] 造 2 个必然失败的用例：一个输出缺字段、一个引用编造行号，确认都被拦下且回喂文本可读。
- [ ] 验证：日志里能看到 `repair=1` 或 `repair=2` 后成功或降级的完整轨迹。
- [ ] commit + 日志。

**会踩的坑**：
1. **校验器自己抛异常把循环炸了**：`validate_final` 内部任何解析都要包进 errors 流程——校验失败是业务信号，不是程序 bug。
2. **回喂文案太笼统**："输出不合格请重试"会让模型原样再犯；错误列表必须具体到字段。
3. **模型把校验错误当新任务**：修 A 字段时顺手改了 B 字段。对策：回喂时附上"其余内容保持不变"，并把上一版 JSON 一起给它。

**自测**：
1. 为什么"格式→契约→语义"这个顺序？反过来做有什么问题？
2. "引用行号真实存在"用程序判定和用 LLM judge 判定，各花多少钱、误差如何？你的答案是哪个？
3. `terminated_by="max_steps"` 的这次运行，最终答案该不该返回给用户？说出你的策略并检查代码里是否一致。

---

## 周五 · JSONL 落盘：给未来 Langfuse 铺轨

**本质**：Agent 的故障 90% 不能靠"再跑一遍"复现——模型输出随机、外部数据会变。唯一的复现手段是把**每一步的完整输入输出**落盘。今天写的 JSONL trace 有两个下游：W12 之前它就是你唯一的调试工具；W21 接 Langfuse/OTel 时，好的字段设计让迁移变成"加一个 exporter"，坏的设计要重跑所有实验。所以字段今天就要按 trace/span 语义定好。

**Java 视角**：你做过全链路日志的 MDC/traceId 透传，这里完全同构：一次会话 = trace，一次 LLM 调用/工具执行/校验 = span。你已经知道没有 traceId 的日志等于没有日志。

**核心讲解**：

一行 = 一个事件（event），一次运行 = 一个文件 `{session_id}.jsonl`。字段设计（这是今天的主体，不是 tracer.py 的代码）：

```json
{"ts": "2026-…T13:45:02.113Z", "session_id": "s-07", "trace_id": "s-07", "step_idx": 3,
 "span_id": "s-07-0003", "parent_span_id": "s-07-0002", "event": "tool_call",
 "tool": "search", "input": {"query": "OOM"}, "output_excerpt": "…截断500字…",
 "output_hash": "sha256:9f…", "latency_ms": 121, "status": "ok", "error": null,
 "model": null, "tokens": null, "cost_est": null}
```

要点逐条讲：

- **`trace_id` / `span_id` / `parent_span_id`**：Langfuse 的 ingestion API 就是这套语义（trace→span/generation 层级），现在用同一命名，W21 时你的 loader 直接映射，不用改一行落盘逻辑。这是"为可观测性选型留后路"的最便宜做法。
- **`step_idx` 单调递增**：断点重放（周六）按它定位；也是幂等键——同一 `session_id + step_idx` 只允许一条，重跑换 session_id，别在原文件里改。
- **`event` 枚举**：`llm_call` / `tool_call` / `validation` / `turn_end` / `error` / `session_start`。周六的 replay 全靠枚举过滤。
- **工具输出双轨**：`output_excerpt`（截断，人读）+ `output_hash`（全文哈希）。全文也存（截断上限设大些，比如 4000 字），因为重放需要**原始输出**当输入；哈希用于判断"两次运行的工具结果是否一致"——数据变了还是模型变了，一眼分清。
- **LLM 事件记 `model`/`tokens`/`latency_ms`/`cost_est`**：cost_est = 单价×token 的粗算，W22 的成本治理直接从这张表出发。
- **敏感信息**：落盘前先过一遍 `redact()`（手机号/密钥/内网 IP 正则替换）。trace 文件以后会进 git 或发给同事复现，**密钥不入仓库**这条铁律延伸到这里。

写入方式：追加模式、每行 `json.dumps(..., ensure_ascii=False)` + `flush()`。别攒到最后一次写——进程崩了全丢，你就白学了今天的课。

**代码**：`experiments/w09/tracer.py`：

```python
import json, hashlib, uuid
from dataclasses import dataclass, field
from datetime import datetime, timezone
from pathlib import Path

@dataclass
class Tracer:
    session_id: str = field(default_factory=lambda: "s-" + uuid.uuid4().hex[:8])
    out_dir: Path = Path("runs")
    _step: int = 0

    def log(self, event: str, *, parent: int | None = None, **payload: object) -> int:
        self._step += 1                    # step_idx 单调递增：重放定位键 + 幂等键
        rec = {"ts": datetime.now(timezone.utc).isoformat(timespec="milliseconds"),
               "session_id": self.session_id, "trace_id": self.session_id,
               "span_id": f"{self.session_id}-{self._step:04d}",
               "parent_span_id": f"{self.session_id}-{parent:04d}" if parent is not None else None,
               "step_idx": self._step, "event": event, **payload}
        self.out_dir.mkdir(parents=True, exist_ok=True)
        with (self.out_dir / f"{self.session_id}.jsonl").open("a", encoding="utf-8") as f:
            f.write(json.dumps(rec, ensure_ascii=False, default=str) + "\n")  # default=str 兜 datetime
            f.flush()                      # 崩了也保得住已跑过的 11 步，攒批写=白学本课
        return self._step

def sha(s: str) -> str:
    return "sha256:" + hashlib.sha256(s.encode()).hexdigest()[:16]

# 用法：sid = tr.log("session_start", question=q)
# tr.log("llm_call", parent=sid, model=M, tokens=usage, latency_ms=..., finish=...)
# tr.log("tool_call", parent=sid, tool="search", input=args, output=obs, output_hash=sha(obs))
```

**今天的动作**：
- [ ] 实现 `Tracer`，在 `react_native.py` 的每个关键点位（session_start/llm_call/tool_call/validation/turn_end）插桩。
- [ ] 跑 3 个问题，产出 3 个 `.jsonl`；用 `jq` 或 `python -m json.tool --compact` 抽查字段完整性。
- [ ] 检查 `runs/` 和 `.env` 一起进了 `.gitignore`（trace 含业务数据，别裸提交；仓库里放一个 `runs/sample.jsonl` 脱敏样例即可）。
- [ ] 验证：随机挑一行 `tool_call`，能仅凭这行回答"输入是什么/输出是什么/多久/成没成功"，不需要翻别的文件。
- [ ] commit + 日志。

**会踩的坑**：
1. **json.dumps 遇 datetime/自定义对象直接 TypeError**：`default=str` 兜住；否则循环中途崩，比模型犯错还难查。
2. **记录的是"当前 messages 引用"而非快照**：list 是可变对象，后面轮次改了它，你落盘的"输入"跟着变——重放时全是假数据。记录时做拷贝或只记本轮增量。
3. **文件攒在内存最后统一写**：死循环触发你手动 Ctrl-C，什么都没了。追加+flush 是纪律。

**自测**：
1. 为什么字段名叫 `trace_id/span_id` 而不是 `session_id/call_id`？说出它和 W21 的具体关系。
2. `output_hash` 和 `output_excerpt` 各自回答什么问题？只有 excerpt 时你会在哪种故障上瞎掉？
3. 同一 session 内 `step_idx` 撞号会发生什么？你的重放逻辑依赖什么不变式？

---

## 周六 · 深度块（120 分钟）：断点重放与一次真实故障定位

**本质**：JSONL 只是日志，能**从任意一步恢复继续执行**才是重放（replay）。这是本周验收的重头，也是 W19 长任务断点续跑的雏形——思想完全一致：状态可序列化 + 步骤可定位 + 副作用可幂等。

### 第一段（55'）：写 replay 工具

**核心讲解**：重放有两种，先分清楚再动手：

1. **纯轨迹回放（trace replay）**：不重新调模型，把第 N 步记录的消息序列还原出来，单步检查/打印。零成本、确定性 100%，回答"当时到底发生了什么"。
2. **断点续跑（resume execution）**：从第 N 步还原状态，**重新调用模型继续往下跑**。回答"如果从那一步改，结果会怎样"。不确定性回来了：`temperature=0` 只能减弱不能消除漂移（provider 端批处理/kv-cache 顺序等导致同输入不保证同输出），笔记里要如实写"重放复现的是**同类**错误而非**同一**字节"。

**幂等恢复的关键设计——按副作用分类工具**：

| 工具类型 | 重放时的策略 | 理由 |
|---|---|---|
| 只读（search/query） | 优先用记录里的输出；`--live` 开关才真重查 | 外部数据可能已变，用记录保证确定性 |
| 写操作（POST/下单/改配置） | **永远不自动重执行**，打印"此处曾执行 X，跳过" | 重放一次 = 线上重复下单一次，事故 |
| LLM 调用 | 重新执行，但输入严格还原 | 它是被测对象本身，缓存它就没意义了 |

**代码**：`experiments/w09/replay.py` 骨架：

```python
import json
from pathlib import Path

def load_events(path: Path) -> list[dict]:
    return [json.loads(line) for line in path.read_text("utf-8").splitlines() if line.strip()]

def find_step(events: list[dict], n: int) -> dict:
    return next(e for e in events if e["step_idx"] == n)

def resume_from(path: Path, n: int, *, live_tools: bool = False) -> None:
    events = load_events(path)
    # 1) 还原到第 n 步的 messages：按 step_idx 顺序重放 llm_call/tool_call 增量（今天主要工作量）
    msgs = reconstruct_messages(events, upto=n)
    # 2) 继续正常循环，但工具层包一层：live_tools=False 直接返回记录的 output；
    #    写类工具无条件跳过执行并打印"此处曾执行 X"——重放一次=线上重复下单一次，事故
    run_loop(msgs, tool_source=events if not live_tools else None)
```

`reconstruct_messages` 是脏活累活：你的落盘事件记录了每轮的 assistant/tool 增量，按 `step_idx` 顺序重放增量即可。**如果这步发现还原不出完整 messages——那是周五的字段设计缺陷，当场补字段重跑，这本身就是本周最有价值的教训之一**，写进笔记。

### 休息（10'）：起身离开屏幕。不刷手机上的技术视频。

### 第二段（55'）：复现一次真实失败 + 定位记录

做一件事：从本周积累的 runs/ 里挑一个**真实失败**的 session（没有就现造：周二文本协议版大概率还你一个解析失败）。流程走满：

1. `grep '"event": "error"'` / 看最后一行 `turn_end` 的 `status`，确认失败形态；
2. `python replay.py runs/s-XX.jsonl --from 5 --print` 逐步看第 5 步的输入输出；
3. 定性：是协议失败 / 工具语义失败（返回空但没报错）/ 模型推理错（输入都对结果错）——三类对应三种修法（改 prompt、改工具、改上下文），笔记里写清"我当时用哪一行证据排除了另外两类"；
4. 修一处，重放验证，`diff` 修复前后轨迹。

笔记落盘 `notes/w09_replay_debug.md`，它就是你第一篇"踩坑复盘"的素材（验收清单第 3 项的公共技术文提前攒料）。

**本周术语卡**

| 术语 | 一句话解释 | 为什么重要 |
|---|---|---|
| workflow | 控制流预定义在代码里的 LLM 编排 | 多数"agent 项目"的真实身份 |
| agent | 模型运行时自主决定步骤与工具使用的循环 | 收益是泛化，代价是可控性 |
| ReAct | Thought→Action→Observation 交替的提示范式 | Agent 循环的历史原型，本周手写两版 |
| 文本协议 / text protocol | 用约定格式的纯文本承载结构化意图 | 脆弱性的来源，对照组 |
| tool calling（原生工具调用） | API 层以 JSON Schema 约束的结构化调用 | 用解码约束消灭语法级错误 |
| constrained decoding | provider 在 token 采样时强制符合 schema | "更稳"的机制解释 |
| 完成判定 / completion validation | 用外部校验替代模型自述"完成" | 终止≠完成，验收硬条款 |
| 降级输出 / graceful degradation | 修复超限后带失败标记返回 | 不隐藏失败是工程素养 |
| JSONL | 每行一个 JSON 对象的追加式日志 | 重放与 W21 trace 的地基 |
| trace / span | 一次会话/一次调用的可观测层级单元 | 字段命名对齐 Langfuse |
| 断点重放 / replay | 从记录的中间状态恢复继续执行 | W19 断点续跑的雏形 |
| 幂等 / idempotent | 重复执行不产生额外副作用 | 写操作重放的安全底线 |

**延伸阅读（可选，通勤时段）**：ReAct 原论文（Yao et al., 2022）的 prompt 附录一节，只看它的格式约定和错误讨论；不用读实验部分。
