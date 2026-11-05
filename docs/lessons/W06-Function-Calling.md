# W06 · Function / Tool Calling

> 本周在计划中的位置：W05 让模型的话变成可校验的对象，本周让这句话变成**动作**。你要在不用任何 Agent 框架的前提下，手写出"模型请求 → 你执行 → 结果回填 → 模型再决策"的循环。后面 LangGraph、MCP、多智能体的所有抽象都是它的换皮，这一周做扎实，那些就只是语法。

## 本周导航
| 日 | 主题 | 今天的产出物 |
|---|---|---|
| 周一 | 工具定义即 JSON Schema，执行权在你 | `src/tools/registry.py` + 一次手工工具调用 |
| 周二 | 单工具循环跑通 | `scripts/one_tool_loop.py`，含每步 messages 打印 |
| 周三 | 多工具与并行调用 | 3 个工具 + 一批 tool_call 的并行分发 |
| 周四 | 错误与终止 | 异常回填 + 步数 / 超时 / 重复调用 / 预算四道闸 |
| 周五 | 写操作安全 | `requires_confirmation` 门 + 审计落盘 |
| 周六 | 抽成 `agent_loop.py` | 可配置循环 + 3 个集成测试（不打真实 API） |
| 周日 | 休息，不许补课 | — |

依赖同 W05（`openai` 1.x、`pydantic` v2、`python-dotenv`）。本周所有工具都用**本地假实现**（写死的 dict），不接真实数据库和外网——本周练的是循环，不是集成。

## 周一 · 工具是什么：一段说明书 + 你的执行权
**本质**：所谓 function calling，模型没有调用任何函数。它在上下文里读到"有这些工具可用"，然后**输出一段符合 schema 的调用意图**（工具名 + JSON 参数）作为回复；要不要执行、用什么凭证执行、结果给不给它看，全在你的代码里。这条边界搞错，后面所有安全设计都会跑偏。

**Java 视角**：模型像一个只会往 MQ 发格式化消息的生产者，你的进程是唯一消费者。它发"我要查天气，city=杭州"，真正去发 HTTP 的是你；它无权提权、无权重试，也看不到你没回填的东西。

**核心讲解**：请求体带 `tools: [...]`，每个元素是 `{"type":"function","function":{name, description, parameters}}`，三个字段权重差别很大：
- **name**：模型唯一能"引用"的标识，必须全局唯一、语义自解释、稳定（改名等于改接口）。字符集通常限字母/数字/下划线/中划线/点，长度上限约 64，以服务商文档为准。
- **description**：模型判断"该不该用这个工具"的**唯一**依据，也是它学参数用法的唯一来源。要写清做什么、不做什么、什么时候别用；上限普遍比 parameters 严（不少服务商限 1024 字符）。
- **parameters**：标准 JSON Schema 对象（不是 Python 类型）。约束的是模型生成的参数结构，机制与 W05 的 structured output 同源——逐 token 屏蔽非法分支，所以 strict 那套限制在这里同样成立。
- **响应侧**：意图在 `resp.choices[0].message.tool_calls`（**数组**，可多条），每条有 `id`、`function.name`、`function.arguments`，其中 `arguments` 是**字符串**，要自己 `json.loads`。`finish_reason` 在发起调用时通常是 `tool_calls`，但判据请用"`tool_calls` 是否为空"，别只看 finish_reason（各家取值有差异）。`tool_choice`：`auto`（默认）/ `none`（禁用）/ `required`（必调但不指定，较新模型才支持）/ `{"type":"function","function":{"name":"query_weather"}}`（指定工具）——**指定工具≠替你执行**，它照样返回意图等你执行。老文档里的 `functions` / `function_call` 是已废弃字段，新模型不支持。

**代码**
```python
import asyncio, json
from typing import Any
from openai import AsyncOpenAI
client = AsyncOpenAI()                  # 异步客户端；同步 Client 放进 async 会阻塞事件循环
WEATHER_TOOL: dict[str, Any] = {        # src/tools/registry.py：模型能看到的该工具的全部信息
    "type": "function",
    "function": {"name": "query_weather",                              # 模型靠 name 引用它
                 "description": "查城市当前天气；不用于历史天气、空气质量、出行建议。",
                 "parameters": {"type": "object", "additionalProperties": False,  # W05 strict 同源
                                "properties": {"city": {"type": "string", "description": "中文城市名，不带「市」字"},
                                               "unit": {"type": "string", "enum": ["celsius", "fahrenheit"]}},
                                "required": ["city"]}}}
async def main() -> None:
    r = await client.chat.completions.create(
        model=MODEL_NAME, tools=[WEATHER_TOOL], tool_choice="auto",
        messages=[{"role": "user", "content": "杭州现在多少度？"}])
    m = r.choices[0].message                    # 意图挂在 message 上，没有独立字段
    for tc in m.tool_calls or []:               # 数组：一次可能给你好几条
        print(tc.id, tc.function.name, json.loads(tc.function.arguments))   # arguments 是 str 不是 dict：本周第一高频坑
asyncio.run(main())
```
**今天的动作**
- [ ] 打开所选服务商的 tool calling 文档，把请求/响应两侧字段名抄成一张对照表进 `docs/`——差异要在第一天暴露，不是周四。
- [ ] 写 `src/tools/registry.py`：`TOOLS: dict[str, dict]`（name → 定义）与 `HANDLERS: dict[str, Callable]`（name → 函数），启动时断言两者键集相等。
- [ ] 用假实现 `query_weather(city, unit="celsius")` 返回写死数据，手工打印一次 `tool_calls` 的完整结构；`tool_choice` 三种取值各跑一次并记录差异。

**会踩的坑**
1. **把 `arguments` 当 dict 用**：`tc.function.arguments["city"]` 直接 `TypeError`。它一定是字符串，而且**可能是畸形 JSON**（模型偶尔塞注释或尾逗号），解析必须 try。
2. **照抄旧版教程**：用 `functions=` / `message.function_call`，旧模型能跑，新模型静默返回 `tool_calls=None`，现象像"模型不肯用工具"。
3. **description 写成人类文档 + 以为指定了 `tool_choice` 就执行了工具**：一句话简介会让模型在多工具里瞎选；而模型只返回意图，你不跑 handler 什么都不会发生，账单照扣。

**自测**
1. 一次请求里，关于这个工具模型实际看到了哪些字节？哪些是你以为它看到了、其实没发出去的？
2. 工具执行失败 / 超时 / 返回空，这三种情况模型分别怎么知道？（提示：只有你回填的它才知道）
3. name 和 description 为什么两个都要约束？只约束一个会漏掉哪类错误？

## 周二 · 单工具循环：把一次调用变成一条链
**本质**：tool calling 只是"模型说它想调工具"。让它**基于工具结果继续回答**，要你手工把结果以 `role="tool"` 的消息塞回 `messages` 再请求一次。这个"发意图 → 执行 → 回填 → 再决策"的 while 循环就是 Agent 的内核，本周把它写对，比学任何框架都值。

**Java 视角**：像手写一个 RPC 闭环——拿请求体、执行、按协议把返回值封回去、再发起调用。封错的报文对端不会报错给你看，而是**继续往下生成**一个看起来很合理的答案，这才是最危险的地方。

**核心讲解**：消息序列必须长成 `system` → `user` → `assistant(带 tool_calls)` → `tool`（每个 tool_call 一条）→ `assistant`（可能又带 tool_calls）→ … → `assistant`(最终文本)。三条硬规则：
1. **assistant 那条带 `tool_calls` 的消息要原样追加回 `messages`**，不能只留 content。漏掉它、或 tool 消息前面没有对应的 assistant 消息，服务商直接 400，错误信息通常提到 tool 消息与前置 assistant 的关联。
2. **tool 消息靠 `tool_call_id` 绑定那次调用**：填 `tc.id`，`content` 必须是**字符串**，dict/list 先 `json.dumps(ensure_ascii=False)`，否则报错或被静默丢。个别服务商还要求一个与工具同名的 `name` 字段（老 API 遗留），带上一般无害，以文档为准。
3. **退出条件是"这次响应没有 tool_calls"**，不是"跑满两轮"：它可能连要三次工具，也可能一次都不要。
4. **打印每步 messages 是今天的重点**：别只看最终答案，把每条消息的 role、有没有 tool_calls、content 前 120 字符打出来。九成"循环不 work"从这张表能直接看出来，最常见一条是 tool 结果里塞了 Python repr（`{'temp': 31}` 单引号）——模型这轮读得懂，下一轮就开始模仿单引号。

**代码**
```python
import asyncio, json
from src.tools.registry import HANDLERS, TOOLS              # 周一的注册表
def log(msgs: list[dict], step: int) -> None:               # 每步看一眼，别等崩了再猜
    print(f"--- step {step}: {len(msgs)} 条 ---")
    for m in msgs:
        names = [c.function.name for c in m.get("tool_calls") or []]
        print(f"{m['role']:<9} tool_calls={names} {str(m.get('content'))[:120]!r}")
async def one_tool_loop(question: str, max_steps: int = 6) -> str:
    msgs: list[dict] = [{"role": "system", "content": SYSTEM_TEXT}, {"role": "user", "content": question}]
    for step in range(max_steps):                            # 上限必须有，见周四
        m = (await client.chat.completions.create(model=MODEL_NAME, messages=msgs,
                                                  tools=list(TOOLS.values()))).choices[0].message
        if not m.tool_calls:                                 # 没有调用意图 = 最终答案
            msgs.append({"role": "assistant", "content": m.content}); log(msgs, step)
            return m.content or ""
        msgs.append({"role": "assistant", "content": m.content,        # 这条必须回填，漏了必 400
                     "tool_calls": [t.model_dump() for t in m.tool_calls]})
        for tc in m.tool_calls:
            try:
                out = await HANDLERS[tc.function.name](**json.loads(tc.function.arguments or "{}"))
            except Exception as e:                            # 异常也封成 tool 消息，别抛穿循环
                out = {"error": type(e).__name__, "detail": str(e)[:200], "retryable": True}
            msgs.append({"role": "tool", "tool_call_id": tc.id,          # id 绑错 = 结果对错了工具
                         "content": json.dumps(out, ensure_ascii=False)})  # content 只能是字符串
        log(msgs, step)
    raise RuntimeError("超过最大步数，循环未收敛")
```
**今天的动作**
- [ ] 用 `query_weather` 把 `one_tool_loop("杭州现在多少度？")` 跑通，并把第 1 步和第 2 步的 `log` 输出并排贴进日志，确认多了 `assistant(tool_calls)` + `tool` 两条。
- [ ] **故意删掉 assistant 那条**重跑一次，亲眼看完错误信息长什么样，抄进 `docs/` 报错对照表；再把 `json.dumps(out)` 换成直接传 dict 跑一次，看是报错还是静默。两条实验各 3 分钟，抵排错 1 小时。

**会踩的坑**
1. **忘了追加 assistant 消息**：`messages with role 'tool' must be a response to a preceding message with 'tool_calls'`。今天必踩，别慌。
2. **多工具时 `tool_call_id` 填错**：模型以为结果对应错了工具，然后开始瞎解释，看起来像幻觉，其实是接线错。
3. **`**args` 里冒出未知参数 / 把结果转成自然语言回填**：模型多给一个 `days=3` → `TypeError`，异常被吞掉后你只看到"答案很蠢"（执行前按 schema 的 `properties` 过滤）；`"杭州 31 度"` 这种回填丢了结构、下游无法复用，要回填 JSON 字符串。

**自测**
1. 不看资料，画出 4 步循环结束时 `messages` 的完整序列（role + 关键字段）。
2. tool 消息为什么要带 `tool_call_id` 而不是靠顺序对应？
3. 模型拿到工具结果后仍然答错，说出两种可能根因（提示：结果太长 / 结果与问题无关）。

## 周三 · 多工具与并行调用
**本质**：模型可以在**一次回复里发起多个 tool_call**（并行调用），也可以在后续轮次串行发起。你的循环必须能处理"一批调用"，逐个回填。真正的难点不是并发执行，而是**工具一多，模型选错的概率上升**——那是定义质量问题，不是代码问题。

**Java 视角**：`tool_calls` 数组像一批并行提交的 `CompletableFuture`：要全部 `get()` 并各自带回执，漏一个回执，下一轮请求就是畸形报文。

**核心讲解**：
- **并行开关**：`parallel_tool_calls: true/false`（多数新模型默认 true）。设 false 后一轮只发一个，轮次变多、总延迟上升，但每步可观测性更好；两种都要实测。注意它与 W05 的严格结构化输出互斥——要 `parallel_tool_calls: false` 才能叠 `json_schema`。
- **并发执行**：同一批 tool_calls 之间无依赖，`asyncio.gather` 更快。回填顺序不影响正确性（靠 id 绑定），但**按原顺序 append** 更利于 diff 和重放。同步 IO 的工具（`httpx.Client`、`sqlite3`）要用 `asyncio.to_thread` 包一层，否则阻塞事件循环——串行时看不出来，并行时直接暴露。
- **注册表模式**：`name → handler` 的 dict 分发，别写 if/elif；启动期断言"定义与 handler 键集相等"，把不一致挡在启动。**硬性习惯：注册表就是白名单**，模型请求未注册的工具名一律拒绝并回填错误，绝不用 `getattr` 动态解析去"猜它想调哪个"。
- **工具变多的三招**：① description 写"只做 X，不做 Y"，Y 往往就是那个和它长得像的工具；② 合并近似工具，用一个 `action` 枚举参数区分，而不是留两个名字相近的工具；③ 分层——先选类别再用类别下的子工具（W17 展开，本周知道有这招即可）。

**代码**
```python
import asyncio, json
from typing import Any
async def search_kb(query: str, top_k: int = 3) -> dict[str, Any]:       # src/tools/builtin.py
    return {"hits": [{"id": f"doc{i}", "text": f"{query} 片段{i}"} for i in range(top_k)]}
async def calc(expr: str) -> dict[str, Any]:                             # 只允许四则运算
    if not all(c in "0123456789+-*/(). " for c in expr):                 # 白名单：绝不把模型输入交给 eval
        return {"error": "illegal_chars", "retryable": True}
    return {"value": eval(expr, {"__builtins__": {}}, {})}                # 教学用；生产换 ast 解析
async def db_query(sql: str) -> dict[str, Any]:                           # 只读账号 + 强制 LIMIT
    if not sql.strip().lower().startswith("select"):
        return {"error": "only_select_allowed", "retryable": False}
    return {"rows": [{"id": 1, "status": "closed"}]}                      # 假实现；同步 IO 工具在此 to_thread
HANDLERS = {"search_kb": search_kb, "calc": calc, "db_query": db_query}
async def run_batch(calls: list[Any], timeout: float = 15) -> list[dict]:  # 一批调用 → 一批回执
    async def one(c):                                       # 各调用之间无依赖，并发跑
        try:
            out = await asyncio.wait_for(HANDLERS[c.function.name](
                **json.loads(c.function.arguments or "{}")), timeout)
        except Exception as e:                              # 未注册、参数错都在这里变结构化错误
            out = {"error": type(e).__name__, "detail": str(e)[:200], "retryable": True}
        return {"role": "tool", "tool_call_id": c.id,        # 顺序与 calls 一致，便于逐次重放
                "content": json.dumps(out, ensure_ascii=False)}
    return list(await asyncio.gather(*(one(c) for c in calls)))
```
**今天的动作**
- [ ] 补齐 3 个工具定义（`search_kb` / `calc` / `db_query`），每个 description 里写一句"不用于…"。
- [ ] 构造一个必然触发并行的问题（"查知识库里连接池的配置，顺便算 05-01 到 05-12 差几天"），观察 `len(tool_calls)`；用 `run_batch` 全部回填跑通。
- [ ] 把 `parallel_tool_calls=False` 重跑一遍记录轮次与耗时差；再加两个描述相近的工具（`search_kb` / `search_docs`）跑 10 次统计选错比例——这个数字就是"工具定义质量"的度量。

**会踩的坑**
1. **只回填第一个 tool_call**：模型下一轮要么重复调用要么放弃，现象像"模型抽风"，实际是报文不完整。
2. **跨工具参数名冲突**：`search_kb(query=...)` 和 `db_query(query=...)` 语义不同，模型会串用；参数名在工具之间也要保持语义一致。
3. **同步工具放进 gather / 给模型开了写工具却没标注**：`httpx.Client`、`sqlite3` 直接调用时 gather 退化成串行还占住事件循环（用 `asyncio.to_thread` 或 `httpx.AsyncClient`）；本周只跑查询类，在模型有可能调出 `DELETE` 之前，门必须已经写好（周五）。

**自测**
1. `parallel_tool_calls=False` 换来什么、付出什么？什么场景该关？
2. 三个工具里模型选错了，你的两种修法分别动的是哪一侧（定义 / 注册表）？
3. 为什么"按原顺序回填"值得专门做？重放调试时它省了什么？

## 周四 · 错误与终止：不让循环烧钱
**本质**：工具会失败，模型会绕圈。一条没有出口的循环能在几分钟内烧掉你一周的预算。所以**每个循环都要有四道闸**：最大步数、单步超时、重复调用检测、总 token 预算。错误处理的原则是"异常也作为 tool 消息回填"——把可修复的错误变成模型的输入，它才有机会改参数重试；抛穿循环，前面几步的钱就白花了。

**Java 视角**：`retry + circuit breaker + timeout` 三件套你熟，只是这里"下一步做什么"的决策交给了模型，而闸门必须握在你手里。异常抛穿相当于把下游 500 直接透传给终端用户；回填错误则像把 `BindingResult` 返回给调用方。

**核心讲解**：
- **错误怎么写给模型**，三种形态差别极大：① 原始 traceback——泄漏内部路径与依赖版本（**安全问题**），且模型不知道该改什么；② 只写"执行失败"——无信息量，模型原样重试，直接进死循环；③ 结构化：`{"error":"validation_failed","detail":"参数 city 不能为空，支持中文城市名","retryable":true}`——模型能据此改参数。默认用 ③；权限类、工具不存在这类不可重试的必须写 `retryable:false`，否则它会一直试。
- **最大步数与单步超时**：步数默认 6–8，到顶就退出，并把"未完成任务 + 已有上下文"作为结果返回上层，而不是抛一个裸异常把上下文丢掉。超时用 `asyncio.wait_for` 包住"一次工具执行"，LLM 请求本身用客户端自带的 timeout、外面再套一层 `wait_for` 处理连接卡死；两类超时分开记（`step_timeout` vs `tool_timeout`），因为诊断路径完全不同。
- **重复调用检测**：对 `(name, canonical_json(args))` 做键并统计次数，同一键出现第 2 次即判定循环。canonical 用 `json.dumps(obj, sort_keys=True)`——否则模型只是换了字段顺序你就检不出来。
- **终止原因必须可观测**：循环返回带 `stop_reason` 的结果对象（`final` / `max_steps` / `loop_detected` / `step_timeout` / `budget`）+ 累加的 `usage`（接着写进 W03 的账本）。别让调用方靠异常类型猜它为什么停。

**代码**
```python
import asyncio, json
from collections import Counter
# LoopResult（自己用 @dataclass 定义，别靠异常区分业务分支）：text / steps / usage / history
#   stop_reason ∈ final | max_steps | loop_detected | step_timeout | budget   ← 终止原因必须显式可观测
def call_key(name: str, args: dict) -> str:         # 闸3 canonical 化：换了字段顺序也算同一次调用
    return f"{name}:{json.dumps(args, sort_keys=True, ensure_ascii=False)}"
async def call_tool(name: str, args: dict, timeout: float) -> dict:     # 白名单：未注册名字绝不动态解析去"猜"
    if name not in HANDLERS: return {"error": "unknown_tool", "retryable": False}
    try:
        return {"ok": await asyncio.wait_for(HANDLERS[name](**args), timeout)}   # 闸2 单步超时
    except asyncio.TimeoutError:                    # ③ 结构化：不回填 traceback，也不只写"失败"
        return {"error": "tool_timeout", "retryable": True, "detail": f"超过 {timeout}s，换更小输入"}
    except Exception as e:                          # 裸 except 会吞 CancelledError，Ctrl-C 就停了
        return {"error": type(e).__name__, "retryable": True, "detail": str(e)[:200]}
# guarded_loop：周二的循环 + 四道闸，回填逻辑一字不改，只在下面标注处插守卫
async def guarded_loop(msgs: list[dict], *, max_steps=6, step_timeout=15.0, max_tokens=20_000) -> LoopResult:
    res, seen = LoopResult(), Counter()
    for step in range(max_steps):                                            # 闸1 最大步数
        r = await asyncio.wait_for(ask(msgs), step_timeout)   # 外层 try 包住：异常→step_timeout 并返回
        res.usage["total"] = res.usage.get("total", 0) + r.usage.total_tokens
        if res.usage["total"] > max_tokens:                                 # 闸4 预算：带上下文退出
            res.stop_reason, res.history = "budget", msgs; return res
        # ...这里接周二写好的 assistant(tool_calls) / tool 回填，原样搬过来...
        for tc in r.choices[0].message.tool_calls:
            args = json.loads(tc.function.arguments or "{}")
            key = call_key(tc.function.name, args)                       # 闸3：按 (工具, 规范化参数) 计次
            seen[key] += 1                                               # 第二次同参 = 绕圈
            # 别写成 seen[key := call_key(...)] += 1：赋值目标的子script 里不允许海象，直接 SyntaxError
            out = ({"error": "duplicate_call", "retryable": False, "detail": "相同参数已调用过，结果不会变"}
                   if seen[key] > 1 else await call_tool(tc.function.name, args, step_timeout))
            if seen[key] > 1: res.stop_reason = "loop_detected"
            msgs.append({"role": "tool", "tool_call_id": tc.id,
                         "content": json.dumps(out, ensure_ascii=False)})
    res.stop_reason, res.history = "max_steps", msgs; return res   # 到顶也返回中间结果，钱没白花
```
**今天的动作**
- [ ] 把周二的循环换成 `guarded_loop`，用 `calc("1/0")` 制造异常，确认异常变成 tool 消息而不是崩循环。
- [ ] 造一个必然绕圈的问题（工具结果里没有答案，模型反复同参调用），验证 `loop_detected` 在**第二次**重复就触发；把 `max_tokens` 设成 3000 跑一次，确认 `budget` 返回带上下文的 `LoopResult` 而非异常。
- [ ] 三种错误文案（traceback / "执行失败" / 结构化）各跑一次，把模型后续行为差异并排贴进日志，写一句结论。

**会踩的坑**
1. **重复检测只比 name 不比参数**：同工具不同参数是正常业务，误判会砍掉正确路径；反过来只比 dict 字面量又不认字段换序。必须用 canonical JSON。
2. **gather 里一个工具失败导致兄弟任务被取消**：`return_exceptions=True` 才能看到"哪个抛了什么"，否则只剩一个异常，其余结果凭空消失。
3. **把 429 当工具错误回填 / 循环不设上限**：限流是服务商侧问题，回填给模型只会让它换措辞再试，照样 429——429 走退避、不进循环逻辑；`max_steps` 与 `max_tokens` 必须有默认值，调用方要显式传大值才能突破。

**自测**
1. 四道闸各拦的是什么？去掉任意一道，你的账单或体验会具体怎么坏？
2. `retryable` 这个字段为什么值得专门为模型设计，而不是给人看？
3. 循环停在 `max_steps` 时，已花的钱换回了什么？怎么让它换回更多？

## 周五 · 写操作安全：门在代码里，不在提示词里
**本质**：查询可以让模型自己跑，写入（改数据、发消息、下单、删文件）必须有人类确认。原因不是"模型笨"，而是**它的输出会不可逆地改变外部世界，而它无法为后果负责**。所以 `requires_confirmation` 是**工具元数据 + 执行层强制门**，不是写进 system 的约定——写进提示词的规则，模型在长上下文里迟早违反。

**Java 视角**：像给方法加 `@RequireApproval` 注解 + AOP 拦截，而不是在业务代码里写"开发者请注意此处需审批"。注解在框架层生效，注释只对你有道德约束力。

**核心讲解**：
- **默认 deny**：每个工具显式标 `read_only: bool`，**未标注或标注缺失一律按写操作处理**。白名单式安全（默认拒绝）比黑名单（默认允许 + 排除）可靠得多——后者只要你漏配一个工具就出事。
- **门的位置**：在 `handler(**args)` **之前**。命中需确认时不执行，而是把这次 `tool_call` 转成**提案**（工具名 + 参数 + 预估影响）向人类取一次 yes/no；yes 才执行并回填结果，no 就回填 `{"error":"rejected_by_user","retryable":false}`，让模型知道要换方案或收口。
- **确认必须是独立的人工输入**：不能用模型自己的输出当确认——它能生成"用户已同意"这种文本。CLI 用 `input()`，Web 场景走独立接口带一次性 token。确认只对**这一次具体参数**有效，参数变了就是新提案（把参数哈希绑进提案 ID）。
- **超时丢弃 + 审计落盘**：提案不能挂着过夜，等待确认要有时限，超时按拒绝处理并结束循环；每次写操作把"提案 + 是否批准 + 实际参数 + 结果"追加进 `logs/tool_audit.jsonl`。密钥、连接串只从 `.env` 读，**绝不进仓库、绝不回填给模型**——工具结果里带出 `password=` 等于把密钥发给了服务商。

**代码**
```python
import asyncio, hashlib, json
from dataclasses import dataclass
from typing import Any, Awaitable, Callable
@dataclass(frozen=True)
class ToolSpec:                                # 定义与它的安全属性绑在一起注册，避免两处漂移
    definition: dict[str, Any]
    handler: Callable[..., Awaitable[Any]]
    read_only: bool = False                    # 默认 False＝要确认；免确认必须显式声明（默认拒绝）
def proposal_id(name: str, args: dict) -> str:  # 绑参数：改一个字段就是新提案，必须重新确认
    return hashlib.sha256(f"{name}|{json.dumps(args, sort_keys=True, ensure_ascii=False)}".encode()).hexdigest()[:12]
async def gate(spec: ToolSpec, name: str, args: dict, *, approve: Callable[[dict], Awaitable[bool]],
               wait_seconds: float = 120) -> dict[str, Any]:   # approve 注入：测试里换假函数
    if spec.read_only:
        return await spec.handler(**args)                      # 只读工具直接执行
    p = {"id": proposal_id(name, args), "tool": name, "args": args}
    try:
        ok = await asyncio.wait_for(approve(p), wait_seconds)   # 超时按拒绝，不留悬挂提案
    except asyncio.TimeoutError:
        ok = False
    with open("logs/tool_audit.jsonl", "a", encoding="utf-8") as f:    # 批准与拒绝都要落盘
        f.write(json.dumps({"proposal": p, "approved": ok}, ensure_ascii=False) + "\n")
    if not ok:                              # 拒绝也要回填：模型据此换方案或收口，而不是又试一次
        return {"error": "rejected_by_user", "retryable": False, "detail": "该写操作未获人工确认，请给出无需写入的替代方案或直接回复用户"}
    return await spec.handler(**args)
async def cli_approve(p: dict) -> bool:        # 唯一合法的人类输入点，不由模型代答
    return input(f"执行写操作 {p['tool']} {p['args']}？[y/N] ").strip().lower() == "y"   # 回车即拒绝
```
**今天的动作**
- [ ] 注册表从"名字 → 函数"改成"名字 → `ToolSpec`"：`search_kb` / `calc` 标 `read_only=True`，新增假写工具 `close_ticket(ticket_id)` 标 `read_only=False`。
- [ ] 跑一次"把这个工单关掉"：先回车拒绝，看模型收到 `rejected_by_user` 后怎么收口；再输入 `y`，确认执行成功且审计文件多了一行含参数哈希的记录。
- [ ] 把 `approve` 换成 `asyncio.sleep(200)` 的假函数，验证超时按拒绝处理、循环能正常退出；顺手检查 `.env` 在 `.gitignore` 里、`.env.example` 只有键名没有值。

**会踩的坑**
1. **把"写入前请征求用户同意"写进 system 当防护**：模型会自己代答"用户已同意"然后继续调工具。防护必须在执行层，与提示词无关。
2. **确认粒度错**：每个只读查询都弹确认（用户第 10 次后就不看了，等于没有门），或一次 `y` 对整个会话生效（越权）。确认只对单个具体参数集有效。
3. **审计只记成功 / 结果里带出凭证**：被拒的提案最有信息量，它暴露模型试图做什么危险动作，两类都要落盘；handler 返回前先脱敏，别指望模型"忽略"它已经看到的密钥。

**自测**
1. `read_only` 的默认值为什么必须是 False？改成 True 会引入什么具体风险？
2. 提案 ID 为什么要把参数哈希进去？不带的话攻击面是什么？
3. 你现在的循环里有几个执行点能绕过 `gate()`？说得出来吗？

## 周六 · 深度块（120 分钟）

### 第一段 55' · 项目实体：`agent_loop.py`
**本质**：把前四天散落的代码收成一个模块：`run_llm_loop(question) -> LoopResult`，参数可配（最大步数、超时、预算、日志、钩子），依赖可注入（LLM 调用、工具注册表、确认器）。**"可注入"是今天的设计核心**——只有把 LLM 调用换成假实现，第二段那三个测试才能不打真实 API、不花钱、可重复。

**核心讲解**：周四的循环搬进来，只改三处：`ask()` 换成注入的 `llm(msgs)`；工具执行一律过 `gate()`；每步把耗时与结果交给钩子。钩子用可调用对象列表而不是继承（`on_step(i, msgs)`、`on_tool(name, args, result, ms)`），**只观测不干预**——在钩子里偷偷改参数或重试，循环语义就不再由参数决定了。`approve` 的缺省值必须是拒绝（`_auto_deny`）：忘了传就是拒绝，绝不能默认放行。这是本周第一次写"给别人用的代码"，W09 的 ReAct 会直接复用它，接口不稳就得回去改。

**代码**
```python
# src/agent/agent_loop.py：对外只暴露 run_llm_loop；循环体 = 周四 guarded_loop + 三处替换
import asyncio, json, time
from collections.abc import Awaitable, Callable
from dataclasses import dataclass, field
from src.agent.guard import LoopResult, call_key                     # 周四的四道闸
from src.tools.registry import REGISTRY, ToolSpec, gate              # 周五的注册表与门
LLM = Callable[[list[dict]], Awaitable[Any]]                    # 注入点：真 SDK 或测试里的假函数
async def _auto_deny(p: dict) -> bool:
    return False                                    # 默认拒绝：漏传 approve 不等于放行写操作
@dataclass
class Hooks:                                                # 钩子只观测，不干预决策
    on_step: list[Callable[..., None]] = field(default_factory=list)
    on_tool: list[Callable[..., None]] = field(default_factory=list)
async def run_llm_loop(question: str, *, llm: LLM, registry: dict[str, ToolSpec] = REGISTRY,
                       max_steps: int = 6, step_timeout: float = 15, max_tokens: int = 20_000,
                       hooks: Hooks | None = None,
                       approve: Callable[[dict], Awaitable[bool]] | None = None) -> LoopResult:
    hooks, approve = hooks or Hooks(), approve or _auto_deny
    msgs = [{"role": "system", "content": SYSTEM_TEXT}, {"role": "user", "content": question}]
    # guarded_loop 的四处替换：① ask(msgs) → await llm(msgs)；② call_tool → await gate(spec, ...)
    # ③④ 每轮开始交 hooks.on_step、每次工具执行前后取 perf_counter() 交 hooks.on_tool（trace 与耗时）
    res = await guarded_loop(msgs, llm=llm, registry=registry, gate_fn=gate, hooks=hooks,
                             max_steps=max_steps, step_timeout=step_timeout, max_tokens=max_tokens)
    res.history = msgs                    # history 是验收时序图的底稿：role 序列要能直接 print 出来
    return res
```
**今天的动作**
- [ ] 建 `src/agent/agent_loop.py`，把 `guarded_loop` 改造成 `run_llm_loop`：`llm`、`approve`、`registry` 全部变成参数，默认值可被测试覆盖。
- [ ] 用真实客户端跑通一次"3 工具 + 1 次写操作确认"的完整对话，确认 `stop_reason=="final"`；加一条 `hooks.on_tool` 把每步工具名、参数、耗时、是否被拒追加到 `logs/agent_trace.jsonl`，字段与 W03 用量账本对齐，W21 可直接读。
- [ ] 打印一次 `res.history` 的 role 序列（如 `sys,user,asst,tool,asst`）贴进日志——这就是阶段 1 验收要画的时序图底稿。

休息 10 分钟（离开屏幕，这段不许省）

### 第二段 55' · 收口：三个集成测试 + 文档
**核心讲解**：测试目标是**路径覆盖**，不是答案质量。用一个假 LLM：把预先写好的响应对象按顺序吐出来，全程不发网络。三个测试各断言什么——① 正常：调过 1 次工具、`stop_reason=="final"`、messages 里出现过 `tool` 角色；② 工具失败：handler 抛异常，断言循环**没崩**且 tool 消息里含 `"error"`；③ 死循环：假 LLM 无限返回同一次调用，断言 `stop_reason=="loop_detected"` **且模型调用次数 ≤ 上限**——后半句才是关键，它证明你真的没在烧钱。别断言自然语言文本：那是你自己写进假 LLM 的字符串，断言它等于什么都没测。

**代码**
```python
# tests/test_agent_loop.py — pytest + pytest-asyncio；一次真实 API 都不打
import json, types
import pytest
from src.agent.agent_loop import run_llm_loop
def fake_llm(script: list[dict]):
    async def _llm(msgs: list[dict]):
        step = script.pop(0)                       # 脚本化：一次调用弹一条预设响应
        tc = [types.SimpleNamespace(id=c["id"], type="function", function=types.SimpleNamespace(
                name=c["name"], arguments=json.dumps(c["args"]))) for c in step.get("calls", [])]
        return types.SimpleNamespace(choices=[types.SimpleNamespace(message=types.SimpleNamespace(
                content=step.get("text"), tool_calls=tc or None))],
            usage=types.SimpleNamespace(total_tokens=120))   # usage 缺了会让预算测试假通过
    return _llm
WEATHER = [{"id": "c1", "name": "query_weather", "args": {"city": "杭州"}}]
@pytest.mark.asyncio
async def test_normal_path(read_only_registry):                  # fixture 只注册 read_only 工具
    res = await run_llm_loop("查天气", llm=fake_llm([{"calls": WEATHER}, {"text": "杭州 31℃"}]))
    assert res.stop_reason == "final" and any(m["role"] == "tool" for m in res.history)
@pytest.mark.asyncio
async def test_tool_error_filled_back(broken_tool):               # handler 抛 ZeroDivisionError
    res = await run_llm_loop("算一下", llm=fake_llm([
        {"calls": [{"id": "c1", "name": "calc", "args": {"expr": "1/0"}}]}, {"text": "无法计算"}]))
    assert res.stop_reason == "final"                              # 关键：循环没被异常打断
    assert '"error"' in [m for m in res.history if m["role"] == "tool"][-1]["content"]
@pytest.mark.asyncio
async def test_loop_detected_caps_the_spend(read_only_registry):
    n = 0
    async def spam(msgs):                                   # 永远返回同一次调用，模拟绕圈
        nonlocal n; n += 1; return await fake_llm([{"calls": WEATHER}] * 10)(msgs)
    res = await run_llm_loop("查天气", llm=spam, max_steps=5)
    assert res.stop_reason == "loop_detected" and n <= 5           # 这条断言不是可选的
```
**今天的动作**
- [ ] 先写假 LLM 工厂并确认它**一次网络都不发**：`pytest -q` 应在 1 秒内出结果（耗时几秒就说明注入没生效）。
- [ ] 三个测试按 正常 → 工具失败 → 死循环 的顺序写通，先正常，另两个才有对照。
- [ ] 把 `run_llm_loop` 的每个参数写进 `src/agent/README.md`（默认值 + 设错会怎样，一行一个），最后 10 分钟给本周打状态，未做项写清"缺什么"，不许写"下周努力"。

**会踩的坑（周六特有）**
1. **测试偷偷打了真实 API**：忘了注入 `llm`，或 `conftest` 加载 `.env` 后走了默认真实客户端。看耗时和账单就能发现——1 秒内跑完才是假的。
2. **断言最终文本**：假 LLM 的文本是你自己写的，断言它等于没断言；要断言**循环行为**（步数、`stop_reason`、消息序列）。
3. **只断言 `stop_reason` 不断言调用次数 / 在钩子里做业务决策**：前者证明不了"少花钱"，`n <= max_steps` 才是这条测试的意义；后者让循环语义不再由参数决定，W09 复用时是排查地狱。另外别把两段 55 压成 110——中途不离开屏幕，第二段质量明显下滑，这条从 W05 起每个周六都在被违反。

**自测**
1. 假 LLM 靠哪些字段和真 SDK 对齐？少给一个（如 `usage`）会让哪个测试假通过？
2. 死循环测试为什么必须断言"模型调用次数"？只断言 `stop_reason` 够吗？
3. 现在把 `agent_loop.py` 交给 W09 的 ReAct 用，接口上最可能缺哪一块？

## 本周术语卡
| 术语 | 一句话解释 | 为什么重要 |
|---|---|---|
| tool calling / 工具调用 | 模型输出"调哪个工具、参数是什么"的结构化意图；旧称 function calling（字段 `functions`/`function_call`） | 执行权在你，模型没有手；旧字段新模型已废弃，抄旧教程必踩 |
| tool definition / 工具定义 | `name` + `description` + `parameters`（JSON Schema） | 模型能看到关于该工具的全部信息 |
| `tool_calls` / `arguments` | 调用意图数组，其中参数是**字符串**形式的 JSON | 每条都要执行并回填；要 `json.loads` 且可能畸形 |
| tool message / 结果回填 | `role="tool"` + `tool_call_id` + 字符串 content | 模型唯一的外部世界信息来源 |
| `tool_choice` | `auto` / `none` / `required` / 指定某工具；并行开关是 `parallel_tool_calls` | 强制指定≠替你执行；并行与严格 schema 互斥，关掉后轮次变多 |
| tool registry / 白名单注册表 | `name → (定义, handler, read_only)` | 未注册工具名一律拒绝，绝不动态解析 |
| loop guard / 循环四道闸 | 最大步数、单步超时、重复调用检测、token 预算 | 少一道就可能一晚上烧光预算 |
| `requires_confirmation` | 写操作执行前必须拿到一次性人工确认 | 提示词里的约定不是门，执行层的才是 |
| dependency injection / 依赖注入 | LLM 调用与确认器作为参数传入循环 | 没它就没法不打真实 API 地测试 |

## 延伸阅读（可选，不占用 1 小时额度）
- 服务商 tool calling 页面的"限制与注意事项"：把工具数量上限、name/description 长度上限、是否支持 `tool_choice:"required"` 抄成一行清单贴进 `docs/`。
- 把今天打印过的某一次完整 `messages` 序列（脱敏后）贴到 `docs/agent-anatomy.md`——W09 讲 ReAct 时你会感谢这张图。
