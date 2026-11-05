# W05 · Prompt 与结构化输出

> 本周在计划中的位置：前四周你能把消息发出去、把用量算清楚；这周解决"拿回来的东西能不能被程序消费"。学完你能用工程手段（而不是改措辞）把一次对话的输出变成可校验、可回放、可对比成本的契约。

## 本周导航

| 日 | 主题 | 今天的产出物 |
|---|---|---|
| 周一 | Prompt 五要素与 system 重写 | `src/prompts/system_v2.md` + 逐条对照表 |
| 周二 | few-shot 的数量、顺序、格式一致性 | `experiments/fewshot/` 的 0/2/5 × 10 次通过率 |
| 周三 | 推理类模型的边界 | 3 任务对照表（通过率 / token / P95 延迟） |
| 周四 | 结构化输出 + 校验失败回喂 | `src/prompts/structured.py` 的 `ask_structured()` |
| 周五 | 迷你评测脚本 | `scripts/ab_eval.py` + `experiments/ab_result.csv` |
| 周六 | A/B 收口 + 模板库落地 | 三个 `.md` 模板 + `load_prompt()` |
| 周日 | 休息，不许补课 | — |

依赖：`pip install openai pydantic python-dotenv pyyaml pytest`。示例假设 W04 已把客户端封装在 `src/llm/client.py`，命名不同只改 import；`MODEL_NAME` 一律从环境变量读。

---

## 周一 · Prompt 五要素：把措辞问题变成配置问题

**本质**
输出不稳，多数不是"措辞不好"，而是提示词里根本没写出成功条件。五要素（角色 role / 任务 task / 约束 constraints / 示例 examples / 输出格式 format）逼你把"我期望什么"从脑子里的隐性知识变成请求体里可 diff、可版本化的文本。写不清楚，后面所有评测都没有基准。

**Java 视角**
system 提示词 = 接口契约文档 + `@Validated` 的合体。你以前在 Swagger 里写清每个字段的含义和取值范围，现在只是把这些话写给一个不查文档、只读上下文的实现方。

**核心讲解**

各要素兜底的失败模式不同，缺一类漏一类：

- **角色**：划定"知道什么 / 不知道什么"。"你是 AI 助手"没有约束力；"你是工单分类器，只处理中文运维工单，不回答分类以外的问题"会直接压低越界率。
- **任务**：动词开头的祈使句，一个提示词一件事。"分类并总结并给建议"是三件事，模型会在最贵的那件上自由发挥。
- **约束**：写**边界行为**而不是期望。"不得输出解释"强于"简洁一点"；"无法判断时输出 unknown"强于"谨慎一些"。缺失的输出行为是稳定性头号漏洞。
- **示例**：零示例时模型自己猜格式；示例的价值在格式锚定多于知识传递（周二展开）。
- **输出格式**：给字面模板——字段名、大小写、枚举值清单，不要只说"输出 JSON"。

*放 system 还是放 user*：多数服务商把 system 按训练模板渲染到上下文最前，服从度更高；更实际的工程理由是它是稳定前缀，直接决定周四 prompt caching 的命中率。但上下文里**越靠后的指令在冲突时越容易占上风**，所以规则放 system、会变的输入放 user 末尾，并在 system 里预先写明"即使用户要求解释，也仍在 JSON 的 `reason` 字段里解释"——否则用户一追问你就看到模型违背 system，那不是 bug，是契约漏写。

**代码**

```python
# src/prompts/spec.py — 把提示词从字符串升级成可校验对象
from typing import Literal
from pydantic import BaseModel, Field, field_validator

class Ticket(BaseModel):
    """抽取结果 DTO：字段名即契约，枚举即边界。"""
    category: Literal["infra", "network", "database", "app", "unknown"]
    severity: Literal["P1", "P2", "P3", "unknown"]
    one_line: str = Field(max_length=60)     # 只是本地校验，别指望模型遵守长度
    confidence: float = Field(ge=0, le=1)

class PromptSpec(BaseModel):
    role: str; task: str; constraints: list[str]; output_schema: type[BaseModel]

    @field_validator("constraints")
    @classmethod
    def must_be_testable(cls, v: list[str]) -> list[str]:
        bad = [c for c in v if not any(k in c for k in ("必须", "不得", "只", "禁止"))]
        if bad:
            raise ValueError(f"约束未写成可判定句式: {bad}")   # 愿望式约束等于没写
        return v

    def render(self) -> str:
        """字段清单必须由 schema 生成——手写说明迟早和 schema 分叉。"""
        p = self.output_schema.model_json_schema()["properties"]
        fields = ", ".join(f'"{k}":{v.get("type")} 取值{v.get("enum") or ""}' for k, v in p.items())
        rules = "\n".join(f"- {c}" for c in self.constraints)
        return (f"# 角色\n{self.role}\n\n# 任务\n{self.task}\n\n# 约束\n{rules}\n\n"
                f"# 输出格式\n只输出一个 JSON 对象，无代码块与前后说明：\n{{ {fields} }}")
```

**今天的动作**
- [ ] 把 W03 的 system 原文按五要素切段，缺哪段补哪段（大概率缺"约束"和"输出格式"）。
- [ ] 写 `src/prompts/spec.py`，`render()` 从 pydantic schema 生成字段清单。
- [ ] 渲染结果存 `src/prompts/system_v2.md`，`git diff` 要看到结构性差异而非同义换词。
- [ ] 同 5 条输入跑 v1/v2 各一次，差异记进日志；收工提交。

**会踩的坑**
1. **约束写成愿望**："尽量简洁"不可判定——上面那个 `field_validator` 就是拦它的；能被单测拒绝的约束才是约束。
2. **一个提示词塞三件事**：评测时你不知道哪一环拖了通过率；本周只做抽取分类。
3. **每轮动态改写 system**（塞时间戳、用户名）：看着无害，周四缓存命中率归零。

**自测**
1. 我的 system 里哪条约束是可判定的？它被违反时哪一行代码会发现？
2. 字段说明为什么必须由 schema 生成？分叉的后果是什么？
3. 输入与 system 都不变、两次输出仍不同——根因在提示词还是采样参数？

---

## 周二 · few-shot：顺序与格式一致性比数量更重要

**本质**
few-shot（少样本示例）不是教模型知识，是**锚定输出分布**：下一个 token 的概率被紧邻的前文强烈影响。给了 3 条 JSON 示例，第 4 次输出 JSON 的条件概率就近似 1；混进一条 Markdown 示例，它就敢混排。所以一致性比数量重要——第 3 条格式歪掉能抵掉前 2 条的一半收益。

**Java 视角**
示例就是给模型的测试 fixture。三个 fixture 里有一个字段命名风格不同，看代码的人会怀疑自己记错了；模型同理，而且它不会向你提 issue。

**核心讲解**

- **数量收益边际递减**：0 → 2 通常肉眼可见（主要是格式合规）；2 → 5 提升很小但输入 token 翻倍、且每条样本每次请求都持续计费；5 → 10 常开始下降——示例互相抢注意力，还容易**泄漏**（把示例里的订单号抄进真实输出）。
- **顺序的影响**：最后一条示例离生成位置最近、权重最大，典型退化是照抄它的某个值。做法是同一批示例打乱再跑 10 次，通过率变动超过 10 个百分点就说明任务对顺序敏感——把最标准的放最后，或把示例从 prompt 挪进 schema 的 `description`。
- **成对结构**：示例要以 `user` + `assistant` 两条消息成对出现，`assistant` 的 content 就是模型该产出的字面 JSON 字符串。把示例当文字描述塞进 system 效果明显差一档——模型学的是"接下来该出现什么"，不是"文档里写了什么"。
- **边界示例比典型示例值钱**：两条"网络故障 → network"没有信息量；一条"原文只说登录不上、其他没提 → unknown"教的是**信息不足时的行为**。至少放一条低信息量 + 一条违反直觉的。同批示例还必须统一缩进、字段顺序、引号风格、枚举大小写；带 `// 注释` 的示例会直接教出非法 JSON。

**代码**

```python
# scripts/fewshot_ab.py — 0/2/5 条示例各跑 10 次，比较通过率
import asyncio, json, time
from collections import Counter
from openai import AsyncOpenAI
from pydantic import ValidationError
from src.prompts.spec import Ticket

client = AsyncOpenAI()   # 必须是异步客户端；同步 Client 放进 async 会卡死事件循环

def build_messages(n: int, case: str) -> list[dict]:
    shots: list[dict] = []
    for text, obj in EXAMPLES[:n]:            # EXAMPLES 自己准备，字段顺序逐条 diff
        shots += [{"role": "user", "content": text},
                  {"role": "assistant", "content": json.dumps(obj, ensure_ascii=False)}]
    return [{"role": "system", "content": SYSTEM_TEXT}, *shots, {"role": "user", "content": case}]

async def one_run(n: int, case: str) -> bool:
    try:
        r = await client.chat.completions.create(
            model=MODEL_NAME, temperature=0.7, messages=build_messages(n, case),
            response_format={"type": "json_object"})    # 只保证合法 JSON，不保证字段
        Ticket.model_validate(json.loads(r.choices[0].message.content))
        return True
    except (ValidationError, json.JSONDecodeError, TypeError):
        return False

async def main() -> None:
    cases = json.load(open("experiments/fewshot/samples.json", encoding="utf-8"))  # 10 条真实工单
    for n in (0, 2, 5):
        t0 = time.perf_counter()
        res = await asyncio.gather(*(one_run(n, c) for c in cases))
        print(f"examples={n} pass={Counter(res)[True]}/{len(res)} {time.perf_counter()-t0:.1f}s")

asyncio.run(main())
```

**今天的动作**
- [ ] 准备 10 条**真实**工单存 `experiments/fewshot/samples.json`（自己业务里的，别造）。
- [ ] 写 5 条 `EXAMPLES`，逐条 diff 确认字段顺序 / 缩进 / 枚举大小写一致。
- [ ] 跑 0/2/5 × 10 次写进 `experiments/fewshot/result.md`；再把第 4、5 条换序跑 10 次记录变动。
- [ ] 一句话结论进日志：我用几条示例，理由是什么。

**会踩的坑**
1. **10 次全过就下结论**：temperature=0.7 下全过说明不了什么，单独再喂一条"信息不足"的输入验边界。
2. **示例带注释或 ` ```json ` 围栏**：模型学出围栏，`json.loads` 当场炸。
3. **忘记示例也算输入 token**：5 条示例每次请求都发，30 条评测下来输入成本可能是 0 示例版的 1.5–2 倍——这笔账周五算。
4. **泄漏**：输出里的订单号原文没有、只在示例出现过 → 把示例的具体值改成占位风格。

**自测**
1. 0→2→5 的通过率曲线什么形状？哪一段最划算？
2. 示例为什么要 `user`/`assistant` 成对，而不是在 system 里描述？
3. 发现模型抄了示例值，两种应对分别是什么？

---

## 周三 · "先想后答"的边界

**本质**
让模型先产出推理再产出答案，本质是**把一次前向的容量换成多次前向的容量**。对多跳、需要中间量、需要自检的任务有效；对"读一段抽三个字段"这种一步映射，它多算了没需要的东西——更慢、更贵，还可能把本来抽对的字段"想歪"。这是算力交换，不是玄学。

**Java 视角**
像给本来 O(n) 能解决的需求套上完整事务 + 审计 + 补偿：重活里保命，轻活里纯 overhead，而且没人替你这次的 P95 买单。

**核心讲解**

两种"推理"必须分清：**提示词层的思维链（CoT / chain-of-thought）**是你在 prompt 里写"先逐步分析再给结论"，适用于普通对话模型，推理文字照样按输出计费；**服务商内建的推理模型（reasoning model）**推理在模型内部完成，账单体现在 `usage.completion_tokens_details.reasoning_tokens`（OpenAI 风格响应里有该字段），多数推理模型拒绝或忽略 `temperature`/`top_p`，控制档位的参数名（`reasoning_effort` / `thinking` / `enable_thinking`）**以你所用服务商文档为准**。混着讲，别人一问就露。

- **该让它先想**：有依赖关系的计算（先算时间差再判断）、约束求解类（SQL 生成、正则构造、多条件筛选——基础错误率高，自检收益大）、多源冲突消解。
- **加了反而差**：纯抽取/分类（一步映射，推理会推动模型"自我说服"，把本该 `unknown` 的硬说成某类）；需要严格 JSON 的链路（推理文本在前、正文在后会稀释格式约束，部分服务商甚至不允许推理模型叠严格 schema）；已在延迟预算边缘的链路。
- **实验设计**：同 prompt、同模型、同采样，只改"是否要求先想"这一项，跑 10 次记**三个数**——通过率、平均 completion tokens、P95 延迟。只看通过率是最常见的自欺。

**代码**

```python
# scripts/reason_probe.py — 3 任务 × 是否 CoT，各 10 次
import asyncio, statistics as st, time
from dataclasses import dataclass
from openai import AsyncOpenAI

client = AsyncOpenAI()
TASKS = {
    "extract": "工单：连接池耗尽，影响 3 个业务方，10:12 恢复。抽 JSON:category/severity/impacted",
    "multi_hop": "订单 05-01 下单、05-04 发货，承诺 72 小时达。是否超时？",
    "sql": "表 orders(id,status,ordered_at)。写 SQL：统计上月已取消订单数。",
}
THINK = "\n\n先逐步推理，再输出最终答案。"

@dataclass
class Row:
    task: str; cot: bool; passed: int; avg_out_tokens: int; p95_ms: float

def heuristic_pass(key: str, body: str) -> bool:
    """判定自己写死：extract 看能否 model_validate，sql 看关键字，multi_hop 看结论词。"""
    ...

async def run(key: str, cot: bool, n: int = 10) -> Row:
    lat: list[float] = []; toks = 0; passed = 0
    for _ in range(n):
        t0 = time.perf_counter()
        r = await client.chat.completions.create(
            model=MODEL_NAME, temperature=0,          # 固定采样，排除随机性干扰
            messages=[{"role": "user", "content": TASKS[key] + (THINK if cot else "")}])
        lat.append((time.perf_counter() - t0) * 1000)
        toks += r.usage.completion_tokens if r.usage else 0
        passed += int(heuristic_pass(key, r.choices[0].message.content or ""))
    return Row(key, cot, passed, toks // n, st.quantile(lat, 0.95))

async def main() -> None:
    print(*await asyncio.gather(*(run(k, c) for k in TASKS for c in (False, True))), sep="\n")

asyncio.run(main())
```

**今天的动作**
- [ ] 三个任务定成"抽取 / 多跳判断 / SQL 生成"，存 `experiments/reason/tasks.json`。
- [ ] 写 `scripts/reason_probe.py`，**先只跑一组**确认出数，别一上来打 60 次请求。
- [ ] 6 组 × 10 次，结果落 `experiments/reason/result.csv`。
- [ ] 在服务商上确认两件事并各写一行结论：`reasoning_tokens` 是否出现在 usage；`temperature` 是否被拒。
- [ ] 结论：项目一里哪些环节开"先想"、哪些明确不开。

**会踩的坑**
1. **同时改两个变量**：加了 CoT 又顺手改 temperature，数据全废；一次实验只允许一个自变量。
2. **推理模型上直接叠严格 `response_format`**：部分组合报 400，或推理文本把 JSON 挤出正文导致解析失败。先查兼容矩阵，别硬试。
3. **只看通过率**：抽取任务加 CoT 后 9/10 → 10/10，但输出 token 40 → 900、P95 1.2s → 8s——这才是"变慢变差"的真实形态。
4. **拿本地小模型的结论外推**：Ollama 上的小模型能力不足，"加 CoT 没用"可能是模型不行的假象。

**自测**
1. 说两类"该先想"和两类"不该先想"的任务，并给判断依据。
2. `reasoning_tokens` 算输入还是输出侧费用？为什么"没看到推理文字"≠"没花推理的钱"？
3. 我的数据里哪组成本涨幅最大？为什么是那组？

---

## 周四 · 结构化输出与"校验失败回喂"

**本质**
`response_format` + `strict: true` 做的事是在**采样阶段把非法 token 的概率置零**——每个 token 只能走在 schema 允许的分支上，于是"JSON 不合法"从高概率事件变成结构上不可能。代价是失去自由文本的灵活性，换回一个可以直接 `model_validate` 的对象。而"结构合法但语义错"（枚举合规但选错、字段填了但内容是废话）schema 管不了，这层只能靠把校验错误回喂重试。

**Java 视角**
strict schema ≈ Jackson 的 `@JsonProperty(required=true)` + 禁用未知字段，只是发生在**序列化那一刻**而非反序列化后。回喂修复 ≈ 把 `BindingResult` 里的 `FieldError` 列表原样返回给调用方重试——你熟的 `@Valid` 那一套。

**核心讲解**

*三档强度*：只说"请输出 JSON"（偶发围栏和多余解释）→ `{"type": "json_object"}`（保证合法 JSON，**不保证字段/枚举/类型**）→ `json_schema` + `strict`（保证符合 schema）。中间那档最容易被误当成契约。原始请求体形状是 `response_format: {"type":"json_schema","json_schema":{"name":"Ticket","strict":true,"schema":{...}}}`，响应仍在 `choices[0].message.content` 里，是字符串。

*strict 的硬性限制*（OpenAI 风格 API 口径，逐条对照你的 DTO）：所有属性必须进 `required`，"可选"用 `"type": ["string","null"]` 表达；每个对象必须 `"additionalProperties": false`；不支持字符串的 `minLength`/`maxLength`/`pattern`/`format`，不支持数字的 `minimum`/`maximum`/`multipleOf`，不支持数组的 `minItems`/`maxItems`/`uniqueItems`；支持 `$defs`+`$ref`、`anyOf`（根节点不能是 `anyOf`）与递归 schema；规模上限约 100 个属性 / 5 层嵌套；**与并行工具调用互斥**，需 `parallel_tool_calls: false`。

这串限制的真实含义是**限制你的 DTO 别太 fancy**：长度、范围、正则全部回到本地 pydantic 校验。两层刚好互补——schema 管结构与枚举，本地管语义与边界。

*pydantic → schema*：`Model.model_json_schema()` 的结果不天然满足 strict（带 `title`、可能漏 `additionalProperties`、可选字段没进 required）。用 SDK 的转换助手（OpenAI Python 1.x 提供 `openai.pydantic_function_tool(Model)`，内部已做严格化）或自己写递归补齐，**别手改生成结果**。

*回喂的要点*：原样重发的成功率约等于掷骰子；把 `ValidationError.errors()` 的精简结构贴回请求，等于给了可操作信号，第二次命中率显著更高。**上限 2 次**——超过 2 次说明 prompt 或 schema 设计有问题，重试只是烧钱，应该抛错逼自己去改提示词。

**代码**

```python
# src/prompts/structured.py — 强制 schema → 本地校验 → 错误回喂重试
import json
from typing import Any, TypeVar
from openai import AsyncOpenAI
from pydantic import BaseModel, ValidationError

T = TypeVar("T", bound=BaseModel)
client = AsyncOpenAI()

def strict_schema(model: type[BaseModel]) -> dict[str, Any]:
    """补齐 strict 两条硬要求：全字段 required + additionalProperties:false。"""
    def walk(node: Any) -> Any:
        if not isinstance(node, dict):
            return node
        node.pop("title", None)
        if (props := node.get("properties")) is not None:
            node["additionalProperties"] = False
            node["required"] = list(props)
        for child in list(node.values()):
            walk(child)                      # 覆盖 properties / items / anyOf 里的子节点
        for key in ("anyOf", "oneOf"):
            for sub in node.get(key, []):
                walk(sub)
        return node
    return walk(model.model_json_schema())

class SchemaRepairFailed(RuntimeError): ...

async def ask_structured(model: type[T], messages: list[dict], *, attempts: int = 2) -> T:
    body = {"type": "json_schema",
            "json_schema": {"name": model.__name__, "strict": True, "schema": strict_schema(model)}}
    convo = list(messages)
    for _ in range(attempts):
        r = await client.chat.completions.create(
            model=MODEL_NAME, messages=convo, response_format=body, temperature=0)
        raw = r.choices[0].message.content or ""
        try:
            return model.model_validate_json(raw)     # 注意：仍是字符串，不是对象
        except ValidationError as e:
            # 回喂结构化错误清单，比自然语言"你输出错了"有效得多
            convo += [{"role": "assistant", "content": raw},
                      {"role": "user", "content": "上面 JSON 未通过校验，只修正后重新输出 JSON：\n"
                       + json.dumps(e.errors(include_url=False, include_input=False), ensure_ascii=False)}]
    raise SchemaRepairFailed(f"{model.__name__} 两次修复仍不合格")
```

**今天的动作**
- [ ] 实现 `strict_schema()`，打印它与 `model_json_schema()` 原始输出的 diff，确认两条硬要求被补齐。
- [ ] `ask_structured(Ticket, ...)` 跑通一次。
- [ ] **故意制造失败**：从 `Ticket` 枚举里删掉一个模型常选的值再跑，观察是否进入回喂分支，最终成功或抛 `SchemaRepairFailed`。
- [ ] `tests/test_structured.py` 用 monkeypatch 假响应锁住"回喂一次即成功 / 两次即抛"两个分支（不打真实 API）。

**会踩的坑**
1. **`strict` 被静默降级**：不支持的服务商会忽略该参数照常返回。查法：故意在 schema 里放一个刁钻枚举值，看输出是否真被限死；若出现集合外取值，说明 strict 没生效。
2. **数字被输出成字符串** `"0.9"`：pydantic v2 严格模式直接拒。修法是在字段描述里写"数字，不要引号"，**不是**在模型侧开 `coerce_numbers_to_str`——那是把契约丢了。
3. **把 `max_length` 写进 schema 指望模型遵守**：strict 不支持长度关键字，会被丢弃，只能本地拦 + 回喂。
4. **修复循环不设上限**：`attempts=2` 是习惯不是建议，`while True` 版本迟早出现在你的账单里。

**自测**
1. JSON mode 与 `json_schema`+strict 各自保证什么、不保证什么？
2. 为什么 strict 要求全字段 required + `additionalProperties:false`？（提示：采样时要能判定下一个 token 的合法集合）
3. 给一个只能被本地校验拦住的错误例子。

---

## 周五 · 迷你评测脚本：让"改了个提示词"变成可验证事件

**本质**
提示词改动只有在**固定样本、固定采样、确定性判分**下才可比。这个脚本就是把三件事固化，产出两个数：通过率、平均 token 成本。没有它，周四的回喂策略、周二的示例条数、周一的措辞调整全是无法讨论的感觉。

**Java 视角**
参数化测试 + 覆盖率报告，只是断言对象是概率性的。`@ParameterizedTest` 你写过上百次，这里是 20 样本 × 2 版本 = 40 次调用，判分器要自己写。

**核心讲解**

- **三个必须固定住的东西**：样本集写死在 `evals/tickets.jsonl`（每条 `input` + 可只含部分的 `expected`），改样本 = 换考卷，旧分数全作废；采样参数要么两边都 0（省事），要么都用线上真实值——用 0 测却上线跑 0.7，你在测一个不存在的产品配置；判分函数确定、无 LLM 参与，抽取就比 `category == expected.category`。本周**禁止引入"LLM 当评委"**（那是 W21），现在引入只会让你分不清是提示词差还是评委差。
- **成本怎么算**：从 `usage.prompt_tokens`/`completion_tokens` 取数按单价折算，`usage` 缺失记 0 并在报告标注。注意 few-shot 示例每条样本都算输入，5 示例版的平均输入 token 可能是 0 示例版的 1.5–2 倍——这条差异只有在评测脚本里才看得见。
- **并发与限流**：40 次串行约 2–3 分钟；用 `asyncio.Semaphore(5)` 收敛，别裸 `gather(40)` 打爆 TPM（每分钟 token 配额），那只会换来一串 429 让通过率假摔。429 要退避重试，且和校验失败分开计数。
- **失败样例必须留原文**：只统计数字的脚本没用。把每条 `input/raw_output/passed` 落盘，收工前 10 分钟人工读失败样例——**本周性价比最高的动作**，它区分"8/20 失败是 JSON 不合法"还是"分类真的错"，两种失败的修法完全相反。

**代码**

```python
# scripts/ab_eval.py — 20 样本 × 2 版本 → 通过率，落 CSV
# 输入 evals/tickets.jsonl，每行 {"input":"...","expected":{"category":"database"}}
import asyncio, csv, json, pathlib
from openai import AsyncOpenAI
from pydantic import ValidationError
from src.prompts.structured import SchemaRepairFailed, ask_structured
from src.prompts.spec import Ticket

client = AsyncOpenAI()
SEM = asyncio.Semaphore(5)                    # 并发上限，别打爆配额

def judge(case: dict, got: Ticket) -> bool:   # 确定性判分：只比关键字段
    return all(getattr(got, k) == v for k, v in case["expected"].items())

async def one(version: str, system: str, case: dict) -> dict:
    async with SEM:
        try:
            obj = await ask_structured(Ticket, [{"role": "system", "content": system},
                                                {"role": "user", "content": case["input"]}])
            passed, raw = judge(case, obj), obj.model_dump_json()
        except (SchemaRepairFailed, ValidationError) as e:   # 契约失败=未通过，类型要可区分
            passed, raw = False, f"<{type(e).__name__}> {e}"
        return {"version": version, "input": case["input"], "passed": passed, "raw": raw}

async def main() -> None:
    cases = [json.loads(x) for x in pathlib.Path("evals/tickets.jsonl")
             .read_text("utf-8").splitlines() if x.strip()][:20]
    versions = {n: pathlib.Path(f"src/prompts/{n}.md").read_text("utf-8")
                for n in ("system_v1", "system_v2")}
    rows = [r for n, s in versions.items()
            for r in await asyncio.gather(*(one(n, s, c) for c in cases))]
    out = pathlib.Path("experiments/ab_result.csv")
    with out.open("w", newline="", encoding="utf-8") as f:
        w = csv.DictWriter(f, fieldnames=list(rows[0])); w.writeheader(); w.writerows(rows)
    for n in versions:
        sub = [r for r in rows if r["version"] == n]
        print(f"{n}: {sum(r['passed'] for r in sub)}/{len(sub)} 通过；失败样例见 {out}")

if __name__ == "__main__":
    asyncio.run(main())
```

**今天的动作**
- [ ] 造 `evals/tickets.jsonl` 20 行，其中 **3 行是"信息不足应为 unknown"**。
- [ ] `judge()` 先只比 `category`，别一上来比四个字段。
- [ ] 跑通 v1/v2 各 20 次，CSV 恰好 40 行。
- [ ] 人工读全部失败行，按"格式错 / 判分错 / 样本本身歧义"三类打标；提交（含真实工单先脱敏）。

**会踩的坑**
1. **样本集太友好**：20 条全是随手挑的典型工单，v1 也 20/20，等于没做实验——差异只在难样本上暴露。
2. **判分器里用 LLM**：分不清提示词退步还是评委抽风，评委的提示词本周不许出现。
3. **把 429 记成失败**：限流错误进 `passed=False` 会让通过率假摔，HTTP 错误与校验错误必须分开计数。
4. **`open()` 忘传编码**：中文环境读到乱码、样本静默损坏。统一用 `pathlib` + 显式 `"utf-8"`。

**自测**
1. 两个版本通过率差多少？在 20 条样本上差几条你才愿意改代码？
2. 成本差异主要来自输入还是输出 token？为什么？
3. 样本从 20 加到 30，哪段代码必须改、哪段绝对不许改？

---

## 周六 · 深度块（120 分钟）

### 第一段 55' · 项目实体：模板库落地

**本质**
把一周的口头成果变成三个月后的你和别人能直接调用的东西：`.md` 文件 + 变量占位 + 一个加载函数。模板不进代码库，下周就会散落在三个脚本里各写一份，然后开始分叉。

**核心讲解**
模板统一放 `src/prompts/`（或 `prompts/`，二选一别混），正文只写文本和 `{{var}}` 占位，元数据用 YAML front matter（`model` / `version` / `task`）。渲染**不用 Jinja2**，本周 `re.sub` 足够——模板里一旦出现 if/for，你就在给提示词写编程语言，没人能 diff 出"这次改了什么"。三个模板对应三类任务（分类 / 抽取 / 改写），每个都保留五要素的分节标题，标题本身就是 checklist。硬规则：**缺变量必须抛错**，不许静默留下 `{{x}}` 进模型——它会把占位符当内容理解。

**代码**

```python
# src/prompts/loader.py — 读 .md → 剥 front matter → 渲染 → 缓存
import functools, pathlib, re
from typing import Any
import yaml
from pydantic import BaseModel, Field

PROMPT_DIR = pathlib.Path(__file__).parent
_VAR = re.compile(r"\{\{(\w+)\}\}")

class Prompt(BaseModel):
    name: str; version: int = Field(ge=1); model: str
    body: str; meta: dict[str, Any] = {}

    def render(self, **vars_: Any) -> str:
        if missing := set(_VAR.findall(self.body)) - set(vars_):
            raise KeyError(f"模板 {self.name} 缺少变量: {sorted(missing)}")
        return _VAR.sub(lambda m: str(vars_[m.group(1)]), self.body)

@functools.lru_cache(maxsize=32)
def load_prompt(name: str) -> Prompt:
    text = (PROMPT_DIR / f"{name}.md").read_text("utf-8")
    meta: dict[str, Any] = {}
    if text.startswith("---"):     # front matter 之后第一段是给人看的标题，不进模型
        _, front, body = re.split(r"^---$", text, maxsplit=2, flags=re.M)
        meta, body = (yaml.safe_load(front) or {}), body.split("\n", 1)[-1]
    else:
        body = text
    return Prompt(name=name, model=str(meta.get("model", "default")),
                  version=int(meta.get("version", 1)), meta=meta, body=body.strip())
```

模板文件 `src/prompts/ticket_classify.md` 的形状（front matter + 五要素分节 + 占位）：

```markdown
---
model: <your-model>
version: 1
task: ticket-classification
---
# 角色
你是中文运维工单分类器，只处理工单分类。
# 任务
阅读 {{domain}} 领域的一条工单原文，输出分类结果。
# 约束
- 不得输出 JSON 以外的字符。
- 必须从 {{categories}} 中选择 category。
- 信息不足时 severity 必须为 unknown，禁止猜测。
# 输出格式
{"category":"...","severity":"P1|P2|P3|unknown","one_line":"...","confidence":0.0}
# 示例
工单：{{example_input}} ／ 输出：{{example_output}}
```

- [ ] 建 `ticket_classify.md` / `data_extract.md` / `text_rewrite.md`，逐个过五要素自查。
- [ ] 写 `load_prompt()` + 一个"缺变量必须抛 KeyError"的单测。
- [ ] 用 `load_prompt("ticket_classify").render(...)` 替换周五脚本里的裸 `read_text` 并跑通。
- [ ] 三个模板各跑 1 条样本确认渲染无事故。

休息 10 分钟（离开屏幕，这段不许省）

### 第二段 55' · 收口：A/B 结论 + README

- [ ] `experiments/fewshot/conclusion.md`：示例条数 × 通过率 × 平均输入 token 的表 + 一句最终选择。
- [ ] `experiments/ab_result.md`：v1 vs v2 通过率、平均成本、失败样例三类占比。
- [ ] 数字不显著就如实写"差异在噪声范围内"。这比编一个结论值钱——W23 的技术文靠的就是这种诚实。
- [ ] `src/prompts/README.md`：命名规则、front matter 字段含义、`load_prompt` 用法、新增模板要走哪几步。
- [ ] 最后 10 分钟给本周打状态（完成 / 部分（缺什么）/ 未做），未做项写清"缺什么"，不许写"下周努力"。

**会踩的坑（周六特有）**
1. **把模板写成人类文档**：给模型看的提示词塞满粗体、表格、装饰性标题，全是白花 token；保留分节结构，去掉排版装饰。
2. **front matter 写了没人读的字段**：元数据只有和评测脚本联动（如按 `version` 选样本集）才有意义，否则是纯负担。
3. **`lru_cache` 让改文件不生效**：调试时改了 `.md` 但进程还缓存旧内容——先排除缓存再怀疑模型。
4. **两段 55 压成一段 110**：中途不离开屏幕，第二段产出质量明显下滑，这是全计划最常被违反的一条。

**自测**
1. `load_prompt` 缺变量为什么必须抛错而不是填空串？
2. 我的 A/B 结论里，质量与成本的取舍点在哪？给一句能对外解释的话。
3. 下周要把模板接进 `agent_loop.py`，现在的 front matter 够用吗？

---

## 本周术语卡

| 术语 | 一句话解释 | 为什么重要 |
|---|---|---|
| system prompt / 系统提示词 | 位于上下文最前、定义角色与规则的消息 | 决定服从度，也是缓存命中的稳定前缀 |
| few-shot / 少样本示例 | 请求里成对附上的"输入→期望输出" | 锚定输出格式；示例越多输入 token 越贵 |
| chain-of-thought / 思维链 | 让模型先产出中间推理再给答案的提示技巧 | 多跳任务的救星，一步映射任务的纯开销 |
| reasoning model / 推理模型 | 服务商内建推理、按推理 token 计费的模型 | 参数兼容性与计费都和对话模型不同 |
| reasoning tokens / 推理 token | 内部推理计入 usage 的输出侧用量 | 看不到≠没花钱，账单位于 usage 里 |
| structured output / 结构化输出 | 用 `response_format` 约束输出符合 JSON Schema | 让输出能被 `model_validate` 直接消费 |
| strict mode / 严格模式 | 采样时屏蔽非法 token，逐 token 保证合法 | 保证结构，代价是全字段 required 等限制 |
| JSON mode | 只保证输出是合法 JSON，不保证字段 | 中间档，最容易被误当成契约 |
| repair retry / 校验失败回喂 | 把 `ValidationError.errors()` 作为反馈重新请求 | 结构错靠 schema，语义错靠它 |
| temperature / 采样温度 | 输出随机性强度 | 评测时必须固定，否则 A/B 无意义 |
| pass rate / 通过率 | 固定样本 + 确定性判分下的成功比例 | 唯一能把措辞改动变成事实的指标 |
| TPM / 每分钟 token 配额 | 服务商的限流维度之一 | 并发调高了只会换来 429，不会变快 |

## 延伸阅读（可选，不占用 1 小时额度）

- 你所用服务商的 structured outputs 页面：把"不支持的 JSON Schema 关键字清单"抄成一页贴在 `experiments/`，改 DTO 时先查它。
- Anthropic《Prompt engineering overview》：对比它对角色 / 约束 / 示例的分节方式，检查你的五要素有没有漏项。
