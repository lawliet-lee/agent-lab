# W10 · LangGraph 状态图

> 本周在计划中的位置：上周你手写了循环、校验、落盘、重放，知道了框架替你管什么；本周正式上 LangGraph，把 W09 的东西迁移成图版本。只学这一个框架是刻意约束——W20 清账日你才有两个项目做参照去补选型表，现在横向比较只会两头都半瓶水。

## 本周导航

| 日 | 主题 | 今天的产出物 |
|---|---|---|
| 周一 | 安装 + 核心概念 + "为什么是图" | 环境跑通 + `notes/w10_why_graph.md` |
| 周二 | 最小图：检索→生成→校验回环 | `experiments/w10/mini_graph.py` |
| 周三 | checkpointer 与 thread_id 会话恢复 | 同 thread 多轮对话演示 |
| 周四 | interrupt 与人工介入 | 写操作确认门（图版） |
| 周五 | 流式：节点事件流 vs token 流 | 终端显示"当前在哪个节点" |
| 周六 | 深度块：W09 全量迁移 + 得失分析 | 图版 ReAct + `notes/w10_migration.md` |
| 周日 | 休息，不许补课 | — |

## 周一 · 为什么需要图，而不是 if-else

**本质**：你的 `agent_loop.py` 就是一个 while 循环——它"能跑"。图（Graph）解决的不是"跑起来"，而是**把控制流从代码里抽出来变成数据**之后附带的一切：状态可持久化（每个节点边界自动快照）、执行可重放/可时间旅行、人工节点可插入（中断-恢复不需要你自己写挂起协议）、流程可视化（一张图贴给同事就能对齐）。if-else 版本这四件事全要你自己造——你上周刚造过一遍，知道造价。

**Java 视角**：两个准确的类比。其一，JDBC 之于手写 socket：LangGraph 不是给你新能力，是把每个项目都要重写的连接管理/事务边界标准化。其二，状态机引擎之于 `switch-case`：转移表是数据时，你才能持久化"当前状态"、才能在任意转移处挂起等人工审批。注意反类比：它**不是** BPMN 那种重量级流程引擎，节点就是普通 Python 函数，没有 DSL。

**核心讲解**：四个概念，一句话各：
- **State（状态）**：一个共享数据结构（`TypedDict` 或 Pydantic `BaseModel`），在节点间流动。关键机制：节点**返回增量**（`{"messages": [新消息]}`），框架用 **reducer** 合并——默认覆盖；字段标注 `Annotated[list, operator.add]` 则追加。你的 ReAct 循环里"把 assistant 消息 append 进 messages"这件事，在这里变成 state 声明的一部分。
- **Node（节点）**：`(state) -> 增量dict` 的函数（也可以是 LLM 调用、工具执行、校验，任何 Python）。节点的粒度就是你的 checkpoint 粒度：**每两个节点边界之间是原子的，边界上是可恢复的**。
- **Edge（边）**：`add_edge("a", "b")` 固定转移。`START`/`END` 是特殊伪节点，标记入口出口。
- **条件边（conditional edge）**：`add_conditional_edges(源节点, 路由函数, [候选目标])`，路由函数读 state、返回下一个节点名（字符串）。**循环就是这么表达的**：`校验失败 → return "retrieve"`，检索节点重新入队，图上出现回边。

和上周对照着记：`should_continue` 就是你的 finish_reason 判断；`while step < MAX` 没了——**最大步数改由 `compile(...)`/invoke 配置里的 recursion_limit 管理**（默认 25，超限抛 `GraphRecursionError`）。这是本周一处真实的控制转移：护栏从你的 while 条件挪到了框架参数，`config={"recursion_limit": 12}` 显式设，别用默认值兜底业务循环。

**代码**：先装环境，再跑通 20 行的最小闭包：
```bash
pip install langgraph          # 主包
pip install langgraph-checkpoint-sqlite   # 周三才用到，但一次装好
```

```python
import operator
from typing import Annotated, Literal, TypedDict
from langgraph.graph import StateGraph, START, END
class State(TypedDict):
    topic: str
    jokes: Annotated[list[str], operator.add]   # add reducer：追加而非覆盖
    verdict: str
def make_joke(state: State) -> dict:
    return {"jokes": [f"关于 {state['topic']} 的第 {len(state['jokes'])+1} 个笑话"]}
def judge(state: State) -> dict:
    return {"verdict": "pass" if len(state["jokes"]) >= 2 else "retry"}  # 占位：真实场景调 LLM 或规则
def route(state: State) -> Literal["make_joke", END]:
    # 路由函数只读 state、返回节点名，不改 state —— 和节点的职责分界
    return "make_joke" if state["verdict"] == "retry" else END
g = StateGraph(State)
g.add_node("make_joke", make_joke); g.add_node("judge", judge)
g.add_edge(START, "make_joke"); g.add_edge("make_joke", "judge")
g.add_conditional_edges("judge", route, ["make_joke", END])  # 回边=循环
app = g.compile()
print(app.invoke({"topic": "Java 程序员", "jokes": [], "verdict": ""}))
```

**今天的动作**：
- [ ] 装包，跑通上面这段，亲眼看 `jokes` 数组被追加了两轮才走到 END（说明回边生效）。
- [ ] 故意把 `judge` 改成永远 retry + 不设限，观察 `GraphRecursionError` 的报错形态——以后见到它就是死循环被护栏掐了。
- [ ] 写 `notes/w10_why_graph.md`：四个能力（持久化/重放/人工节点/可视化）各自"if-else 版你自己怎么造、造价多少"，对照上周你真实写过的那部分。
- [ ] 验证：笔记里每条能力都能指认 W09 里的对应手工实现（如持久化↔JSONL、重放↔replay.py）。能→commit + 日志。

**会踩的坑**：
1. **节点返回了整个 state 而不是增量**：`return state` 会把全部字段再过一遍 reducer，list 字段直接翻倍。只返回改了的 key。
2. **忘了标注 reducer，list 被覆盖**：不写 `Annotated[..., operator.add]` 时默认语义是**后写覆盖**，你的 messages 永远只剩最后一条——症状是"模型完全不记得上一轮"。
3. **在路由函数里改 state**：返回值只是字符串，你的 `state["x"]=...` 静默丢失。改状态是节点的活。
4. **`add_conditional_edges` 第三参写成 dict 映射但 key 和路由返回值对不上**：运行时报 unknown node，编译期不检查。今天先用 list 形式。

**自测**：
1. 用一句话说清 node 边界为什么等于 checkpoint 边界。
2. reducer 解决什么问题？没有它的话，两个并行节点写同一字段会怎样？
3. `recursion_limit` 和你上周手写的 `MAX_STEPS` 语义差别在哪（提示：数的是什么——轮数还是节点执行次数）？

## 周二 · 最小图：检索 → 生成 → 校验，失败回检索

**本质**：任务表指定的这个图是"带自愈回环的 workflow"（用上周一的判据自测：路径可枚举、循环出口由校验器决定而非模型——它是 workflow，LLM 只是节点内的工人）。今天的目标不是难，是**把图的生命周期看全**：每步状态变化、回环怎么走、终止怎么走。

**Java 视角**：像带重试的策略链：fetch → transform → validate，validate 失败回到 fetch 重新取（比如换个查询词）。区别是重试逻辑长在图结构里而不是代码里，所以它是可看见、可持久化的。

**核心讲解**：三节点一条件边。`retrieve` 节点调你的假检索（W06 的即可，真实项目 W11 再上），并把命中片段写入 state；`generate` 节点带着片段调 LLM 出答案；`check` 节点做**确定性**校验（引用是否存在、长度、必填段落——就是 W09 周四 `validate_final` 的图化）。路由函数按 `check` 写入的 verdict 决定回 `retrieve` 还是走 END。

设计上要注意两点，都是"图 vs 循环"的差异点：

1. **重试信息放 state，不放闭包**。回检索时为什么要重查？把"上次校验失败原因"写进 state，`retrieve` 读它来调整查询词（比如拼上失败原因里的关键词）。state 是节点间唯一的通信通道——养成这个习惯，周四它就是你 resume 时的依据。
2. **尝试次数也要进 state**（`attempts: int`），别依赖 recursion_limit 兜底回环。recursion_limit 是熔断器，`attempts >= 2 → END` 才是业务语义的"最多修两次"。两层都要。

**代码**：`experiments/w10/mini_graph.py`（LLM 部分你从 `llm/` 层拿现成的，伪略处已注释）：
```python
import operator
from typing import Annotated, Literal, TypedDict
from langgraph.graph import StateGraph, START, END
from pydantic import BaseModel, ValidationError
class GraphState(TypedDict):
    question: str
    docs: list[str]                 # 每轮整体覆盖：检索结果本来就该是新的
    draft: str
    errors: Annotated[list[str], operator.add]
    attempts: int
def retrieve(state: GraphState) -> dict:
    query = state["question"]
    if state.get("errors"):
        # 把失败原因折进查询：上一轮"缺少来源引用"→这轮多取候选片段
        query = f"{query}（补充依据：{state['errors'][-1][:80]}）"
    return {"docs": fake_search(query, top_k=3), "attempts": state.get("attempts", 0) + 1}
def generate(state: GraphState) -> dict:
    ctx = "\n---\n".join(state["docs"])
    return {"draft": llm_answer(state["question"], ctx)}   # 你的 llm/ 层函数
class Answer(BaseModel):
    text: str
    source_indexes: list[int]
def check(state: GraphState) -> dict:
    errs: list[str] = []
    try:
        ans = Answer.model_validate_json(state["draft"])
    except ValidationError as e:
        return {"errors": [f"输出不符合契约: {e.errors()[:2]}"]}
    if any(i < 0 or i >= len(state["docs"]) for i in ans.source_indexes):
        errs.append("引用了不存在的片段索引，请重新检索并只引用现有 0..%d" % (len(state["docs"]) - 1))
    if len(ans.text) < 20:
        errs.append("答案过短，疑似未基于检索片段")
    return {"errors": errs} if errs else {"errors": []}
def route(state: GraphState) -> Literal["retrieve", END]:
    if state["errors"] and state["attempts"] < 3:
        return "retrieve"
    return END     # 注意：错误没解决也放行——带 errors 的降级输出交给调用方
b = StateGraph(GraphState)
for name, fn in [("retrieve", retrieve), ("generate", generate), ("check", check)]:
    b.add_node(name, fn)
b.add_edge(START, "retrieve"); b.add_edge("retrieve", "generate"); b.add_edge("generate", "check")
b.add_conditional_edges("check", route, ["retrieve", END])   # 回边=自愈循环，组装全部显式写出不许略
app = b.compile()
for chunk in app.stream({"question": "连接池耗尽的默认超时是多少？", "docs": [], "draft": "", "errors": [], "attempts": 0}, stream_mode="updates", config={"recursion_limit": 12}):
    print(chunk)   # 每个节点执行完打印 {节点名: 状态增量}，回环轨迹一目了然
```

（若你装到的版本 stream 输出是 `{"type": ..., "data": ...}` 信封格式而非 `(节点名, 增量)`——`# 当前版本写法请核对官方 tutorial`，先原样打印一个 chunk 再写消费逻辑，以能打出"每步谁改了哪些字段"为准。）

**今天的动作**：
- [ ] 完整写出三节点+条件边组装（上面省略的部分自己补，不许复制依赖理解）。
- [ ] 用 `app.stream(..., stream_mode="updates")` 跑 2 个问题：一个一次通过、一个故意让 `fake_search` 先返回空触发至少 1 次回环。
- [ ] 把回环那次每步的 `attempts`/`errors` 变化抄进笔记，标注"回边在第几步被触发、在第几步放弃"。
- [ ] 验证：能仅凭打印输出给别人讲清这张图的执行轨迹，不翻代码。
- [ ] commit + 日志。

**会踩的坑**：
1. **`generate` 拿到旧 docs**：`retrieve` 失败分支提前 return、没更新 docs。每个出口都检查返回的 key 集合。
2. **回环无限但 recursion_limit 到才停**：route 里只判 `errors` 忘了判 `attempts`。症状：同样错误重复 25 次。两层熔断缺一不可。
3. **降级输出的语义没定义**：END 走到时 state 里 errors 非空，你 invoke 返回值到底算成功还是失败？今天就定下来：返回 state 全量、由调用方看 `errors`，图自己不当哑巴。

**自测**：
1. `check` 为什么用 pydantic 而不用 LLM？今天这一版哪个环节将来值得换成 LLM judge，为什么仍然要加 attempts 上限？
2. 把错误原因写进 state 而不是直接重新检索，给回环带来了什么？
3. 这张图是 agent 还是 workflow？用 W09 三条判据逐条答。

## 周三 · checkpointer 与 thread_id：会话自动恢复

**本质**：checkpointer 让图在**每个节点边界**把 state 快照写入存储；`thread_id` 是会话主键。同一 `thread_id` 下再次 invoke，框架自动把上次终态作为本次起点——你上周用 JSONL+replay.py 手工实现的"恢复"，这里是一个构造参数的事。而且它恢复的不只是消息历史：图走到哪、哪个节点挂起、还差什么，都在快照里。

**Java 视角**：`thread_id` 之于 checkpointer ≈ HttpSession 的 session ID 之于容器持久化的 session（Tomcat 把 session 写 Redis/文件那种）。checkpoint 本身类似事务日志里的快照点：不只有"当前值"，还带元信息（哪个节点写的、第几步），为周四的"从任意点挂起/恢复"和 time-travel 留好了结构。

**核心讲解**：
- **装谁**：本地开发用 SQLite checkpointer（任务表指定）。包名 `langgraph-checkpoint-sqlite`，import `from langgraph.checkpoint.sqlite import SqliteSaver`，用 `SqliteSaver.from_conn_string("checkpoints.sqlite")` 拿到，**它要实现上下文管理器协议**（`with` 里用；离开 with 连接关闭）。老文档里出现的 `MemorySaver` 在新版官方文档里叫 `InMemorySaver`（`from langgraph.checkpoint.memory import InMemorySaver`）——进程内存、重启即丢，只配写单测。`# 当前版本写法请核对官方 tutorial`：拿到包后先跑一段 with 代码确认导入路径没变。
- **编译时机**：`checkpointer` 在 `compile(checkpointer=...)` 传入，不是 invoke 时传。传了它的图**必须**在 config 里给 `thread_id`，不给直接报错——框架拒绝"有状态但不指定谁的状态"这种含糊。
- **恢复机制**：invoke(thread_id=T) 时：读 T 的最新 checkpoint → 作为初始 state 合并（消息列表经 reducer 追加）→ 从 START 跑。所以"多轮对话"= 同 thread_id 连续 invoke，你的 `messages` reducer 自动把历史带上。
- **快照读取**：`app.get_state(config)` 拿当前终态（`values`/`next`/`config`），`app.get_state_history(config)` 拿全部历史快照列表。周四的 resume、以及"W21 之前唯一的检查手段"都靠它们。
- **thread 不是用户**：`thread_id` = 一个会话/一条对话线，用户 ↔ thread 的一对多映射是你业务的事（`user:{id}:chat:{uuid}` 这种拼法够用）。换 thread_id 就是干净的新会话，**测试隔离也用它**。

**代码**：`experiments/w10/chat_graph.py`（两节点极简对话图 + 持久化）：
```python
import operator
from typing import Annotated, TypedDict
from langchain_core.messages import AnyMessage, HumanMessage  # 消息类型沿用你W06起的依赖
from langgraph.graph import StateGraph, START, END
from langgraph.checkpoint.sqlite import SqliteSaver
class ChatState(TypedDict):
    messages: Annotated[list[AnyMessage], operator.add]
def respond(state: ChatState) -> dict:
    reply = llm_chat(state["messages"])          # 你的 llm/ 层，返回 AIMessage
    return {"messages": [reply]}
builder = StateGraph(ChatState)
builder.add_node("respond", respond)
builder.add_edge(START, "respond")
builder.add_edge("respond", END)
with SqliteSaver.from_conn_string("checkpoints.sqlite") as saver:
    app = builder.compile(checkpointer=saver)
    cfg = {"configurable": {"thread_id": "demo-01"}}   # 会话主键在这
    app.invoke({"messages": [HumanMessage("我叫张三，记住。")]}, cfg)
    app.invoke({"messages": [HumanMessage("我叫什么？")]}, cfg)  # 没带历史，但答得出
    snap = app.get_state(cfg)
    print(len(snap.values["messages"]), snap.next)      # 4 条消息；next 为空=已走完
```

第二个 invoke 只传了一条新消息，模型却知道"张三"——历史来自 SQLite 里的快照，不是来自你传的输入。这个反差就是今天的验收点。

**今天的动作**：
- [ ] 跑通上面双 invoke，确认第二轮不带历史也能答对；`sqlite3 checkpoints.sqlite ".tables"` 看一眼落盘结构。
- [ ] 换一个新 thread_id 重复第二轮，确认答不出（证明隔离生效，不是模型瞎猜）。
- [ ] `get_state_history(cfg)` 打印该 thread 的全部快照，找出"respond 节点执行前"的那个 checkpoint。
- [ ] 验证：向自己解释清楚"messages 变 4 条"的完整链路（谁写入、谁合并、谁落盘）。
- [ ] commit；`checkpoints.sqlite` 加进 `.gitignore`（里面是业务对话数据，硬习惯同密钥不入仓库）。

**会踩的坑**：
1. **每轮 invoke 都手动把历史传全**：又叠加上了 checkpointer 的自动历史 → 消息翻倍、token 暴涨、模型开始复读。持久化图 + 增量输入，二选一你的记忆逻辑：接了 checkpointer 就**只传新消息**。
2. **thread_id 每次随机生成**：等于永远没有恢复。它必须来自业务（会话 ID），且跨进程稳定——这正是明天"跨重启继续"的前提。
3. **with 块外调用 app**：saver 连接已关，报 closed database。图的 compile 依赖 with 作用域，CLI 工具就整个生命周期包一层。
4. **改了图结构后旧 checkpoint 加载"成功"但行为诡异**：state 模式变了而快照没清，`# 字段级兼容行为请实测确认`。开发期遇到怪状态先删 sqlite 文件重跑，排除这个变量再怀疑模型。

**自测**：
1. checkpointer 和上周 JSONL 的本质区别？（提示：一个记录"发生了什么"供事后分析，一个保存"现在是什么"供继续执行。）
2. `get_state(cfg).next` 非空意味着什么？（图挂起/中断在哪些节点——周四的主角。）
3. 为什么说"接了 checkpointer 仍手动拼历史"是双倍付费？从 token 和正确性两方面说。

## 周四 · interrupt 与 resume：把人工确认做进图里

**本质**：agent 执行写操作（改配置、删数据、发工单）前需要人来点头。你在 W06 周五手写过确认门——命令行 `input()` 阻塞等待。那是**进程内挂起**：终端一关、状态全丢。LangGraph 的 `interrupt()` 是**持久化挂起**：节点抛中断信号 → 图停住、快照落盘 → 进程可以退出 → 任何时候（换一台机器也行）带着 `Command(resume=...)` 回来继续。等待审批三小时和等待三秒钟，机制上完全一样。

**Java 视角**：W06 的 `input()` 像把 HTTP 请求线程 park 住等业务人员回来点按钮——无线程占用上限、重启即失效。`interrupt`+checkpointer 的组合是工作流引擎的"任务挂起-待办-签收"模型：状态在库里，进程无状态，任意实例可恢复。你做过审批流就应该秒懂：**interrupt 生成的 payload 就是待办单，Command(resume) 就是签收动作**。

**核心讲解**：

用法三件套（均已按当前官方文档核实）：`from langgraph.types import interrupt, Command`；节点函数体内调 `interrupt({"question": ..., "options": ...})`——它不返回值，第一次执行时**抛出特殊异常使整图停下**；恢复时对同一 thread 执行 `app.invoke(Command(resume=人的答复), config)`，节点**从头重新执行**，这次 `interrupt()` 调用处直接返回 resume 的值，代码继续往下。

**最重要的一条机制细节**：恢复时该节点从开头再跑一遍。意味着——interrupt 之前的副作用会重复执行。所以写操作节点的铁律是：**interrupt() 之前不得有不可逆副作用**。"先记录审计日志/扣库存，再问用户确不确定"在这个模型下是错的；顺序必须先是纯读（拼审批信息），interrupt，然后才写。

对照你手写的确认门，框架替你省的三件事：(1) 挂起状态的序列化（你以前要么阻塞要么自己造 pending 文件）；(2) 审批超时/换人审批的恢复路径（新的进程读同一 thread 即可）；(3) 中断点可查询——`get_state(cfg).next` 直接告诉你图卡在哪个节点，`.tasks[*].interrupts` 里躺着审批 payload。没省的：审批人是谁、通知渠道、超时升级——业务逻辑照旧你写。

**代码**：`experiments/w10/hitl_graph.py`：
```python
from typing import TypedDict
from langgraph.graph import StateGraph, START, END
from langgraph.checkpoint.memory import InMemorySaver   # 演示用；跨进程就换 SqliteSaver
from langgraph.types import Command, interrupt
class S(TypedDict):
    action: str
    approved: bool
    log: list[str]
def prepare(state: S) -> dict:
    return {"log": [f"准备执行: {state['action']}"]}    # 纯计算，无副作用
def gate(state: S) -> dict:
    # payload 面向审批人：干什么、影响面、拒绝会怎样，一次说全
    decision = interrupt({
        "question": f"即将执行危险操作【{state['action']}】，确认？",
        "options": ["yes", "no"],
    })
    # ↑ 首次执行到这里图就停了；resume 后这行直接拿到 Command(resume=) 的值
    return {"approved": decision == "yes"}
def write_op(state: S) -> dict:
    if not state["approved"]:
        return {"log": ["用户拒绝，未执行"]}
    return {"log": ["副作用已执行"]}                      # 不可逆操作只出现在 interrupt 之后
b = StateGraph(S)
for name, fn in [("prepare", prepare), ("gate", gate), ("write_op", write_op)]:
    b.add_node(name, fn)
b.add_edge(START, "prepare"); b.add_edge("prepare", "gate")
b.add_edge("gate", "write_op"); b.add_edge("write_op", END)
app = b.compile(checkpointer=InMemorySaver())
cfg = {"configurable": {"thread_id": "ops-42"}}
out = app.invoke({"action": "重启生产数据库", "approved": False, "log": []}, cfg)
print(out["__interrupt__"])                # invoke 正常返回，中断信息在 __interrupt__ 键里
ans = input("审批人答复 yes/no > ")         # 演示用同步等待；真实场景这里可以是几小时后另一进程
out2 = app.invoke(Command(resume=ans), cfg)
print(out2["log"])
```

**今天的动作**：
- [ ] 跑通 interrupt→resume 全流程，打印 `get_state(cfg).next` 在挂起前后的值（挂起时是 `("gate",)`——图明确告诉你卡在哪）。
- [ ] 演示跨进程恢复：挂起后 Ctrl-C 杀掉进程，另起脚本用 SqliteSaver + 同 thread_id 执行 `Command(resume="yes")`，确认 write_op 照常执行。这一步做成，你对持久化中断的理解才算落地。
- [ ] 把 `write_op` 里一行注释测试副作用：在 gate 的 interrupt **之前**加一行 `print("写了一半")` 或模拟副作用，resume 后观察它执行了几次，结论写进笔记。
- [ ] 写 3 行对比：W06 手写门 vs 今天的门（阻塞行为/跨进程能力/状态查询能力三列），一并 commit + 日志。

**会踩的坑**：
1. **resume 到一半副作用翻倍**：就是上面测试的那次——interrupt 前的代码重跑。症状常是"重复发邮件/重复插库"。纪律：节点里 `interrupt` 之前只准纯读。
2. **resume 用错 thread_id**：Command 给了新会话，图在旧 thread 上永远挂着。恢复前 `get_state_history` 确认。
3. **把 interrupt 当 if 条件用**：`if interrupt(...) == "yes"` 里嵌复杂表达式没问题，但别在条件两侧放副作用调用——重跑语义会让你怀疑人生。
4. **忘了图必须带 checkpointer**：没 checkpointer 的图 invoke 返回的是错误而不是中断挂起——无状态可挂。

**自测**：
1. 为什么 `interrupt()` 首次执行时"不返回而是抛出"，resume 时"直接返回"？用节点重跑解释这个双重行为。
2. 审批 payload 里该放哪些信息？（要操作的资源、变更内容、回滚方式、影响面。）缺一样，审批人会在哪个环节骂你？
3. W19 长任务断点续跑和今天的关系：哪些设计你今天已经用上了（thread 语义、幂等分区）？

## 周五 · 流式：节点事件流 ≠ token 流

**本质**：两种"流"回答不同的问题。**节点事件流**（`stream_mode="updates"`）：图跑到哪了、每步改了什么——给进度条和 trace 用。**token 流**（`stream_mode="messages"`，LLM 增量）：答案文本一个字一个字出来——给打字机效果用。它们的粒度、频率、数据结构全不同，把两码事混在一个 for 循环里是这个模块最经典的困惑源。

**Java 视角**：像 SSE 事件流里的 `event: progress`（结构化里程碑）和 `event: delta`（内容增量）两种事件类型；前端对它们的渲染逻辑完全不同。你的终端就是前端，今天学会分辨 event 类型。

**核心讲解**：
- `stream_mode="updates"`：每执行完**一个节点**产出一次 `{节点名: 状态增量}`。一个 10 步循环的 agent 大约几十个事件，低频、结构化，正是"当前在哪个节点"的数据源。
- `stream_mode="values"`：每个节点后推**全量 state 快照**。简单但每帧都是整个 messages 列表，费带宽，前端做整体刷新才用。
- `stream_mode="messages"`：产出 `(LLM消息增量, metadata)` 二元组，节点内 `model.stream` 级别的 token，高频（一次生成几百个）、无结构。metadata 里有节点名/tags，**可以用来把 token 归属到节点**。
- **可以组合**：`stream_mode=["updates", "messages"]`，此时每个 chunk 是 `(模式名, 数据)` 二元组，按模式分流处理——今天的终端 UI 就靠这个。
- **注意版本差异**：官方 tutorial 最新示例出现了带 `version="v2"` 的信封式 chunk（`chunk["type"]`/`chunk["data"]`）。你装到的版本用哪种形态，**先打印一个原始 chunk 看结构再写渲染逻辑**，`# 当前版本写法请核对官方 tutorial`——这是本讲义里唯一不能逐字给你保证的 API 点。

一个真实产品的组合姿势：updates 驱动"⏳ 正在检索…"这类阶段提示，messages 只在你**决定把某个生成节点的输出直接透传给用户**时消费。工具执行节点永远不该出现在 token 流里。

**代码**：`experiments/w10/stream_ui.py`（沿用周三的 ChatState 图，给 respond 换成流式模型输出）：
```python
from langgraph.checkpoint.sqlite import SqliteSaver
NODE_LABEL = {"respond": "✍️ 生成回答", "retrieve": "🔎 检索中", "check": "🧪 校验中"}
with SqliteSaver.from_conn_string("checkpoints.sqlite") as saver:
    app = builder.compile(checkpointer=saver)     # 用你周二/周三组装好的图
    cfg = {"configurable": {"thread_id": "stream-demo"}}
    streaming = app.stream(
        {"messages": [HumanMessage("用三句话总结连接池调优要点")]},
        stream_mode=["updates", "messages"],       # 组合模式
        config=cfg,
    )
    for chunk in streaming:
        mode, data = chunk       # 若你的版本返回 v2 信封（dict），改为按 data["type"] 分流
        if mode == "updates":
            for node_name in data:
                print(f"\n[{NODE_LABEL.get(node_name, node_name)}]", flush=True)
        elif mode == "messages":
            msg_chunk, meta = data
            if getattr(msg_chunk, "content", ""):
                print(msg_chunk.content, end="", flush=True)   # token 级透传
    print()
```

跑起来的预期效果：先打印 `[✍️ 生成回答]`（节点开始），随后答案逐字出现——**这两件事在数据层面的来源完全不同**，你要能说出来：前者是 updates 模式在节点**完成**时产出（节点边界事件），后者是 messages 模式在模型**解码中**产出。严格说 updates 的"当前在哪个节点"其实是"刚跑完哪个节点"；要做真·实时当前节点，用 values 模式的起点帧或给节点加 custom 事件。这个细微差别写进笔记。

**今天的动作**：
- [ ] 换成组合模式跑通"阶段提示 + 打字机"终端输出：先原样打印 5 个 chunk 的结构存进笔记，再写渲染——顺序别倒。
- [ ] 打印一个 messages 模式的 metadata，确认里面节点名字段叫什么（`langgraph_node`？），token 归属能按它过滤。
- [ ] 验证：中途 Ctrl-C 再重跑同 thread_id，确认（有 checkpointer）会话没烂——流式消费失败 ≠ 图状态损坏。
- [ ] commit + 日志。

**会踩的坑**：
1. **把 updates 当 token 流渲染**：每个事件是个 dict 不是文本，你 `print(chunk, end="")` 打出满屏花括号。两种模式的数据类型先打印再消费。
2. **token 流里 AIMessage 的 tool_calls 增量 content 为空串**：模型在"说话"前先"决定调工具"，这段流你看到的就是空，别误判卡死——配合 updates 看心跳。
3. **终端没 flush**：`print(..., end="")` 不加 flush，行缓冲让你以为没输出。
4. **在流式循环里做重活（落库/再调 LLM）**：消费速度拖住 TCP 读取，表现成"流变卡"。流式回调里只做渲染，持久化交给 checkpointer 和 trace。

**自测**：
1. 用户抱怨"半天没反应"，但 token 其实一直在流。该升级到哪种模式排查，看什么数据？
2. updates 事件和 W09 你 JSONL 里的 llm_call 事件，信息量差在哪？（谁记了什么、谁丢了什么——W21 接 Langfuse 时谁补上？graph 的 updates 天然缺 latency/tokens，要在节点内自测补记。）
3. 为什么 messages 模式能拿到 token 而 updates 不能？从产出时机（解码中 vs 节点完成）回答。

## 周六 · 深度块（120 分钟）：W09 全量迁移 + 得失分析

**本质**：把 W09 的 `react_native.py`（模型循环 + 校验 + JSONL trace）迁成 LangGraph 图。产出两件：图版可运行代码、一份**得失清单**——这份清单是阶段 2 验收第二条（"能说出各自代价"）的直接证据，也是你面试里"为什么选/不选框架"那题的答料。

### 第一段（55'）：迁移实施

节点划分照这个走（别自由发挥，先对齐你上周的结构才能逐项对比）：

- `agent_turn`：调模型（带 tools），返回 assistant 消息进 `messages`。
- `route`：`finish_reason=="tool_calls"` → tools；有最终内容 → validate；否则 END。
- `tools`：遍历 `tool_calls`，执行、`ToolMessage(tool_call_id=...)` 回填。W06 的 requires_confirmation 门在这升级为图结构：写类工具路由到 `confirm` 节点（interrupt），确认后才进 `tools`——你上周的确认门和今天的挂起恢复在一张图上会师了。
- `validate`：W09 周四的 `validate_final`，失败且修复轮<2 回 `agent_turn`（回喂错误消息）。
- `tracer`：把 W09 的 JSONL Tracer 包成节点，每轮记 `llm_call/tool_call` 事件。**字段别丢**：这是你回放能力的保底，框架 checkpoint 不能替代 trace（checkpoint 存"现在是什么"，trace 存"当时发生了什么+为什么"，两回事）。

骨架（关键行，其余你补）：

```python
class ReactState(TypedDict):
    messages: Annotated[list[AnyMessage], operator.add]
    errors: Annotated[list[str], operator.add]
    repairs: int
    observations: list[str]        # 供 validate 核对引用
def route(state: ReactState) -> Literal["tools", "confirm", "validate", END]:
    last = state["messages"][-1]
    if getattr(last, "tool_calls", None):
        return "confirm" if any(tc["name"] in WRITE_TOOLS for tc in last.tool_calls) else "tools"
    if isinstance(last.content, str) and last.content:
        return "validate"
    return END
b = StateGraph(ReactState)
# add_node: agent_turn/tools/confirm/validate/tracer；边按上面划分接
app = b.compile(checkpointer=saver)
# invoke 时 config 同时给 thread_id（会话恢复）和 recursion_limit=16（护栏，替代 MAX_STEPS）
```

行为等价性自查：跑 W09 那 10 道题，通过率、平均步数、token 成本三列对比上周数据——**不该显著变好或变坏，变了要能解释为什么**（比如校验回环从 while 变图回边，错误回喂的消息条数可能不同）。

### 休息（10'）

### 第二段（55'）：得失分析——给你一个评估框架

别写"框架真好用"。用这张五行表，逐行回答"上周我手写 X，现在 X 由谁提供、质量如何、我付出了什么"：

| 维度 | 手写版（W09） | 图版（今天） | 判定 |
|---|---|---|---|
| 消息组装粒度 | 完全自由，逐 token 可控 | 节点内仍自由，但增量合并语义由 reducer 定 | 损失？ |
| 错误处理/重试 | 每处 try/except 自己写，策略全见 | 节点级重试可配置，跨节点补偿仍自写 | 混合 |
| 成本/步数熔断 | while 上限+预算 break，语义自定 | recursion_limit 统一数"节点执行次数"，预算要自己埋 | 注意语义漂移 |
| 持久化与恢复 | JSONL 手写、replay.py 手工还原 | checkpointer 免费，含中断点 | 明显收益 |
| 人工介入 | input() 阻塞，进程死即失效 | interrupt 持久挂起，跨进程恢复 | 明显收益 |

**分析框架就三条追问**（写进笔记的骨架，比结论重要）：

1. **等价性追问**：手写版的每个控制（重试语义/熔断/消息拼装），框架版有对应钩子吗？没有的那个，我还能拿回来吗（节点内自写）？——能拿回的是"被抽走的样板"，拿不回的是"全局拦截点"（比如框架在节点间加了什么你没见过的合并逻辑，你只能观测不能改动）。
2. **观测性追问**：排障时，手写版我 grep 自己的 JSONL 就行；框架版我除了 trace 还要理解 checkpoint 存储结构和 stream 信封格式——**认知面扩大了多少**？（这就是代价：黑盒从 1 个循环变成 1 个运行时。）
3. **锁定点追问**：trace/replay/confirm 这些能力是跟着我的代码走的；thread_id+checkpointer 的会话模型是跟着 LangGraph 走的。W21 的 Langfuse 接入、W19 的断点续跑，哪些会变成"只有这个框架才给得舒服"？依赖深度=迁移成本，今天先记录、不评价（W20 再算总账）。

我的预判给你对照（允许你推翻，但要拿数据推翻）：**收益最大的两项是持久化中断（周四）和团队可沟通性（贴一张图 vs 讲一个 while）；损失最大的一项是对"节点间时刻"的独占控制**——手写版你可以在任意两行代码间埋熔断、记 trace、改消息，框架版这些只能发生在节点边界（即"checkpoint 边界"），粒度从语句级降到节点级。这个粒度换来的是免费快照语义，交易是否划算取决于你的循环有多长——节点内塞一整坨逻辑（大节点）等于自愿退回手写并多付一层抽象，这是最常见的 LangGraph 反模式。

**收尾清单**（第二段后半，很枯燥但必须做）：图版目录整理进 `experiments/w10/agent_graph/`；两版各留一个可复跑入口 `python -m ... --q "..."`；笔记 commit；把本周术语回看 10 分钟。

**本周术语卡**

| 术语 | 一句话解释 | 为什么重要 |
|---|---|---|
| StateGraph | 以共享状态为中心组装节点与边的构造器 | 一切的入口 |
| node | `(state)->增量` 的函数，原子执行单元 | 节点边界=checkpoint 边界 |
| edge / conditional edge | 固定转移 / 路由函数动态选下一站 | 回边表达循环与自愈 |
| state reducer | 增量合并规则（覆盖/追加），Annotated 声明 | 忘标就"失忆" |
| checkpointer | 每步快照状态的存储插件（内存/SQLite/PG） | 会话恢复与中断的基座 |
| thread_id | 会话主键，绑定一条 checkpoint 时间线 | 给错等于没恢复 |
| interrupt / resume | 节点内持久挂起 / Command(resume=) 唤醒 | 人工审批的正确形态 |
| 节点重跑 | resume 时所在节点从头执行 | 副作用必须在 interrupt 之后 |
| stream_mode | updates/values/messages 等产出粒度 | 两种流混学是困惑之源 |
| time travel | 从历史 checkpoint 分支重执行 | W19 断点续跑的现成能力 |
| 大节点反模式 | 一个节点塞整个循环 | 退回手写还多付抽象层；调试第一工具是 get_state(_history) |

**延伸阅读（可选）**：官方 docs 的 "Why LangGraph" / durability 概念页各扫一遍（通勤），只带一个问题读：它还给了哪些我今天没用到的能力（并行分支、子图）——记两个名字即可，W17 多智能体会用到。
