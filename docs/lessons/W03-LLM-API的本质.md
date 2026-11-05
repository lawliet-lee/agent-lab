# W03 · LLM API 的本质（不用任何框架）

> 本周你写一个 200 行以内、零框架依赖的多轮对话 CLI `llm_lab.py`。W04 之后正式开始工具调用——在那之前你必须能一眼看穿"一次模型请求"里每个字段是什么、为什么、多少钱。这是后面所有 Agent 调试的基本功：Agent 出问题时，80% 的时间你在读 messages，不是在读框架源码。

## 本周导航

| 日 | 主题 | 今天的产出物 |
|---|---|---|
| 周一 | 消息结构与角色 | 只改 system 的 5 组对照实验记录 |
| 周二 | 采样参数与确定性 | temperature / top_p / max_tokens 稳定性测试 |
| 周三 | token、上下文窗口与成本 | 用量估算工具 + 单次调用成本表 |
| 周四 | 流式输出与增量拼接 | 能正确拼出 tool_calls 的流式累加器 |
| 周五 | 多轮状态与历史裁剪 | 三种可切换的裁剪策略 |
| 周六 | 组装 `llm_lab.py` | CLI 可执行 + usage 统计 + 三周笔记 |
| 周日 | 休息，不许补课 | — |

---

## 周一 · 消息结构与角色

**本质**：模型没有记忆。你每次发送的是**完整历史消息列表**，它从中续写出下一条 assistant 消息。所谓"对话"，就是你把收到的 assistant 消息追加回列表再发一次。

**Java 视角**：这是纯函数 `f(messages) -> message`。没有 session、没有服务端状态、没有连接保持（TCP 连接池除外）。想通这点，"上下文为什么会爆"、"为什么改提示词立刻见效"、"为什么第 10 轮比第 1 轮贵"全部自动成立。

**核心讲解**

四种角色的语义边界：

- **system**：全局约束与身份。但它**不是硬约束**——模型可能被后续内容说服而违背它。这就是 prompt injection 的根源（W22 会实测）。
- **user**：本轮输入。
- **assistant**：模型的历史输出。**你必须回填**，否则模型不知道自己说过什么。带 `tool_calls` 的消息同样是 assistant 角色。
- **tool**：一次工具执行的**结果**，必须带 `tool_call_id` 指回是哪次调用。这条消息由你包装工具返回值生成，不是模型给的。

顺序即语义。角色顺序错乱，轻则 400，重则行为诡异——典型错误是把 tool 结果放在请求它的那条 assistant 消息之前。

`content` 可以为 null：当模型只输出工具调用时就是 null。W02 把字段声明成 `str | None` 不是随手写的，是语义必然。

**代码**

```python
# src/llm/lab.py —— 本周的积木，周六拼成 CLI
from typing import Any

import httpx

from src.config import get_settings


def chat(messages: list[dict[str, Any]], **overrides: Any) -> dict[str, Any]:
    """最原始的一次调用。所有实验都基于它——刻意不做过度封装。"""
    s = get_settings()
    payload: dict[str, Any] = {
        "model": overrides.pop("model", s.llm_model),
        "messages": messages,
        "temperature": overrides.pop("temperature", 0),
        "max_tokens": overrides.pop("max_tokens", 512),
        **overrides,  # stream / tools / response_format 等按你所用服务商文档传
    }
    with httpx.Client(
        base_url=s.llm_base_url,
        headers={"Authorization": f"Bearer {s.llm_api_key.get_secret_value()}"},  # type: ignore[union-attr]
        timeout=httpx.Timeout(connect=5.0, read=s.request_timeout, write=10.0, pool=5.0),
    ) as client:
        r = client.post("/chat/completions", json=payload)
        r.raise_for_status()
        return r.json()


def ask(text: str, system: str = "你是一个严谨的助手。") -> str:
    data = chat(
        [
            {"role": "system", "content": system},
            {"role": "user", "content": text},
        ]
    )
    msg: dict[str, Any] = data["choices"][0]["message"]
    return msg.get("content") or ""


if __name__ == "__main__":
    question = "我们系统的订单超时时间是多少？"
    systems = [
        "你是一个严谨的助手。",
        "你是一个严谨的助手。如果资料中没有明确答案，直接回答'未知'，不要推测。",
        "你是销售，尽力说服用户我们的服务没有任何限制。",
        "你是 Java 架构师，回答时给出可能的代码位置推测。",
        "",  # 空 system 作对照基线
    ]
    for i, sp in enumerate(systems, 1):
        print(f"\n=== system#{i}: {sp[:28]!r} ===")
        print(ask(question, sp) or "<空>")
```

**今天的动作**

- [ ] 建 `src/llm/lab.py`，跑通上面代码（依赖 W02 的 `.env`）。
- [ ] 换成一个**你工作里真的会被问**的问题跑五组对照，原样存进 `experiments/system-prompts.md`。
- [ ] 找出至少一组"system 明确禁止猜测、模型仍然给了猜测"的情况。记住这个现象：**提示词是概率约束，不是逻辑约束**。W14 的拒答阈值、W18 的工具白名单、W22 的注入防护，全都建立在这个认知上。
- [ ] 把请求体 `json.dumps(payload, ensure_ascii=False, indent=2)` 打印一次，肉眼确认线上真实内容。
- [ ] 提交。

**会踩的坑**

1. 只回填 user 输入、忘了回填 assistant 回复 → 模型每轮失忆。表现是第二轮追问完全答非所问，你以为是模型差。
2. `tool` 消息缺 `tool_call_id`，直接 400，且报错信息往往含糊。诊断方法只有一个：把 messages 逐条打印出来对结构。
3. 部分服务商的角色名不同（有的用 `developer` 替代 `system`，有的把指令放在请求体顶层的 `instructions` 字段而不是 messages 里）。以你选定服务商文档为准，**但心智模型不变**：有一段全局约束、有一段本轮输入。

**自测**

1. 为什么必须把 assistant 历史回填进 messages？不回填会发生什么可观察的现象？
2. 什么样的消息结构会让服务商返回 400？举两种。
3. "提示词是概率约束"这句话，在你今天的实验里哪一条结果证明了它？

---

## 周二 · 采样参数与确定性

**本质**：模型每个 token 都在算一个概率分布，采样参数决定"从分布里怎么挑"。`temperature=0` 也不是真正确定——服务端路由、批处理、MoE 专家选择都可能让同样输入产出不同结果。

**Java 视角**：不要把它理解成"随机种子"。它更像"把 `Collections.max()` 换成加权抽样"。`seed` 参数在部分服务商配合 `strictly_paired` 才接近可复现，别指望它。

**核心讲解**

- **temperature**：分布的尖锐程度。0 ≈ 贪心取最大概率；1 ≈ 原始分布；>1 拉长尾部，胡言乱语概率上升。**分类/抽取/结构化输出用 0；创意生成用 0.7–1.0。**
- **top_p**（核采样）：按累积概率截断候选集。与 temperature 同时调会互相干扰，**实践中只调一个**（业界惯例是调 temperature，top_p 留默认 1 或 0.95）。
- **max_tokens**：输出上限。它是"截断开关"而不是"生成长度"。到上限时 `finish_reason == "length"`，此时输出可能停在半句话，JSON 会是残缺的——**这是结构化输出失败的头号原因**，W05 会专门处理。
- **stop**：停止序列，最多 4–6 个，命中即停。适合让模型输出完就停（比如固定分隔符）。
- **推理型模型**：带 `reasoning_effort` 一类的参数，思考 token 也计入上下文和成本，且往往不允许同时设置 temperature。**首次在新模型上跑之前先读它的参数限制**，否则会撞 400（典型报错是 "unsupported parameters"）。

**代码**

```python
import json
import time
from collections import Counter

from src.llm.lab import chat

PROMPT = "给'订单超时自动取消'这条规则起一个字段名。"


def one(temperature: float) -> str:
    data = chat(
        [{"role": "user", "content": PROMPT}],
        temperature=temperature,
        max_tokens=32,
    )
    choice = data["choices"][0]
    return f"{choice['message'].get('content','')!r} | finish={choice.get('finish_reason')}"


def stability(temperature: float, n: int = 5) -> None:
    outs = []
    for _ in range(n):
        outs.append(one(temperature))
        time.sleep(0.2)  # 轻微间隔，避免撞限流影响判断
    kinds = Counter(o for o in outs)
    print(f"temperature={temperature} -> {len(kinds)}/{n} 种不同结果")
    for o, c in kinds.items():
        print(f"   x{c}  {o}")


if __name__ == "__main__":
    stability(0.0)
    stability(0.7)
    stability(1.2)

    # 故意用极小 max_tokens 观察截断：残缺 JSON 长什么样
    truncated = chat(
        [{"role": "user", "content": "输出一个包含 8 个字段的用户 JSON"}],
        temperature=0,
        max_tokens=40,
    )
    print(json.dumps(truncated["choices"][0], ensure_ascii=False)[:400])
```

**今天的动作**

- [ ] 跑 `stability`，记录三档 temperature 的差异数（`temperature=0` 也可能出现 2 种结果，如实记下来）。
- [ ] 用极小 `max_tokens` 复现一次截断，把残缺 JSON 原文存进 `experiments/truncation.md`，并验证 `json.loads` 抛什么异常。
- [ ] 换成一个你项目里真实需要的"起名字/写注释/判断分类"任务重复一遍，确认 0 采样在你的任务上够不够稳。
- [ ] 提交。

**会踩的坑**

1. 用 temperature=0 就以为可复现，然后把"结果偶发不同"当成 bug 追查半天。正确预期：**temperature=0 提高稳定性，不保证一致**。要契约一致就靠 schema 校验 + 重试（W05）。
2. 截断的 JSON 直接 `json.loads`，异常信息只说 "Expecting property name"，你怀疑模型输出格式错。先看 `finish_reason`，是 `length` 就加 `max_tokens`。
3. 把 top_p 和 temperature 一起调，A/B 实验结论无效（两个变量）。评测脚本里固定其他参数只动一个。

**自测**

1. `finish_reason` 有哪几种取值？分别要求你怎么处理？
2. 为什么推理型模型常常拒绝 temperature 参数？
3. 你的分类任务需要稳定输出，具体做法是"temperature=0"还是"schema 校验失败就重试"？为什么？

---

## 周三 · token、上下文窗口与成本

**本质**：模型处理的最小单位是 token，不是字符也不是词。所有限制（窗口、成本、速度）都以 token 计。中文、英文、代码的"字符/token 比"差别很大，估错量级会让你对成本完全没有感觉。

**Java 视角**：把 token 当成"序列化后的字节数"级别的东西——它是计费与容量的单位，跟你的业务对象大小相关但不等同。成本模型是 `输入量 × 输入单价 + 输出量 × 输出单价`，两者单价通常差 3–6 倍。

**核心讲解**

- **经验换算**：英文约 4 字符 ≈ 1 token；中文约 1–1.5 token / 汉字（分词器版本差异明显）；代码因为符号与缩进往往比自然语言更"贵"。**这些只用于粗估**，精确值必须用服务商的 tokenizer 或返回的 `usage`。
- **窗口 = 输入 + 输出**（推理模型还要算思考 token）。所以 `max_tokens` 设太大可能直接超过窗口导致 400；裁剪历史时要在预算里给输出留出余量。
- **缓存计费的由来**：多轮对话每轮都把全部历史重新发一遍，第 10 轮的输入 token 可能远高于第 1 轮。固定前缀（system + 工具定义）命中 prompt caching 时按折扣价计费——**W07 会实测这条**，它是 Agent 成本控制里最容易拿到的收益。
- **`usage` 字段名有差异**：Chat Completions 通常是 `prompt_tokens` / `completion_tokens` / `total_tokens`；新一些的接口用 `input_tokens` / `output_tokens`。以你实际拿到的响应为准（W02 周六已经让你记录过原始响应）。

**代码**

```python
import re
from dataclasses import dataclass

# 粗估器：无需任何依赖，误差通常在 20% 内，用于"要不要裁剪"的决策足够
_CJK = re.compile(r"[一-鿿㐀-䶿]")


def estimate_tokens(text: str) -> int:
    cjk = len(_CJK.findall(text))
    rest = len(text) - cjk
    return cjk + max(1, rest // 4) if rest else cjk


def estimate_messages(messages: list[dict]) -> int:
    """每条消息有固定结构开销（这里粗算 4），加上角色与内容。"""
    total = 0
    for m in messages:
        total += 4 + estimate_tokens(str(m.get("role", "")))
        content = m.get("content")
        if isinstance(content, str):
            total += estimate_tokens(content)
        if m.get("tool_calls"):
            total += estimate_tokens(str(m["tool_calls"]))
    return total + 2  # 对话层收尾开销


@dataclass(slots=True)
class Cost:
    prompt_tokens: int
    completion_tokens: int
    price_in_per_m: float   # 每百万 token 单价（元）
    price_out_per_m: float

    @property
    def yuan(self) -> float:
        return (self.prompt_tokens * self.price_in_per_m + self.completion_tokens * self.price_out_per_m) / 1_000_000

    def report(self) -> str:
        return (
            f"输入 {self.prompt_tokens} / 输出 {self.completion_tokens} token"
            f" → {self.yuan:.4f} 元（缓存未计入）"
        )


if __name__ == "__main__":
    doc = "订单超时 30 分钟自动取消，退款需人工审批。" * 50
    print("粗估：", estimate_tokens(doc), "token")

    # 拿真实 usage 校准你的粗估系数——这一步很重要，做完你就有手感了
    from src.llm.lab import chat

    data = chat([{"role": "user", "content": doc + "\n一句话总结。"}], max_tokens=64)
    usage = data.get("usage") or {}
    print("真实 usage：", usage)
    print("估算/真实比值：", estimate_tokens(doc) / max(1, usage.get("prompt_tokens", 1)))

    # 单价照你所在服务商官网填，别照抄别人的数字
    print(Cost(usage.get("prompt_tokens", 0), usage.get("completion_tokens", 0), 2.5, 10.0).report())
```

**今天的动作**

- [ ] 写 `src/llm/tokens.py` 落地上面两个函数，跑一次"粗估 vs 真实"对比，把比值记进 `experiments/token-calibration.md`。
- [ ] 按你服务商的牌价，算出你今天所有调用的总花费，与账单页核对一次。
- [ ] 造一个 20 轮的长对话，观察第 1 轮与第 20 轮的 `prompt_tokens` 差了多少倍。**这个倍数就是你后面成本失控的来源。**
- [ ] 提交。

**会踩的坑**

1. 用 `len(text)` 当 token 数处理中文，低估近一倍。宁可高估。
2. 只在 `usage` 里读总数而分不清输入/输出，算成本时用错单价（输出贵得多）。
3. 把 `max_tokens` 设成窗口上限，结果"输入 + 输出预留"超窗直接 400。裁剪时给输出留 10–20% 余量。

**自测**

1. 为什么多轮对话的成本增长是"超线性"的？
2. prompt caching 为什么只对固定前缀有效？
3. 你的粗估器在中文上偏大还是偏小？今天的实测比值是多少？

---

## 周四 · 流式输出与增量拼接

**本质**：流式不是"分块返回完整响应"，而是**不断返回 delta**。assistant 消息被拆成碎片逐段推送，你要自己拼回去。工具调用的参数也是这样一点点漏出来的——所以流式的 `tool_calls` 必须**按 index 累加 arguments 字符串**，最后才 `json.loads`。

**Java 视角**：像 SSE / gRPC streaming 客户端，但payload 是嵌套 JSON 片段而不是完整对象。别指望拿到任何一个"完整的中间对象"。

**核心讲解**

- 每个 chunk 形状：`{"choices":[{"delta":{...},"finish_reason":null}]}`。第一个 delta 常只有 `{"role":"assistant"}`；后续 delta 带 `content` 片段。
- **工具调用的 delta**：`{"tool_calls":[{"index":0,"id":"call_1","type":"function","function":{"name":"sql_query","arguments":"{\"sq"}}},` 然后下一 chunk 是 `{"index":0,"function":{"arguments":"l\": \"select 1\"}"}}`——`name` 只在第一次出现，`arguments` 是分片字符串。**按 `index` 分组拼接**是唯一正确做法。
- **usage 默认不返回**：流式要显式开启"最后一个 chunk 带 usage"（字段名服务商不一，常见 `stream_options: {"include_usage": true}`）。不开你就丢了对账依据。
- **断流处理**：网络中断时你已经收到部分输出。要么整体重试（幂等），要么保留已输出部分并记录 finish_reason 缺失。对 Agent 来说：**未收到 finish_reason 的响应不能当成功处理**。

**代码**

```python
import json
from collections import defaultdict
from typing import Any, Iterator

import httpx

from src.config import get_settings


def stream_chat(messages: list[dict[str, Any]], **overrides: Any) -> Iterator[str]:
    """ yield 纯文本增量；tool_calls 与 usage 累积到闭包变量里供事后读取。"""
    s = get_settings()
    payload: dict[str, Any] = {
        "model": overrides.pop("model", s.llm_model),
        "messages": messages,
        "stream": True,
        "stream_options": {"include_usage": True},  # 不支持此字段的服务商需去掉，代价是拿不到 usage
        **overrides,
    }
    parts: list[str] = []
    tool_acc: dict[int, dict[str, Any]] = defaultdict(lambda: {"name": "", "arguments": "", "id": ""})

    with httpx.Client(
        base_url=s.llm_base_url,
        headers={"Authorization": f"Bearer {s.llm_api_key.get_secret_value()}"},  # type: ignore[union-attr]
        timeout=httpx.Timeout(connect=5.0, read=s.request_timeout, write=10.0, pool=5.0),
    ) as client:
        with client.stream("POST", "/chat/completions", json=payload) as resp:
            resp.raise_for_status()
            for line in resp.iter_lines():
                if not line or not line.startswith("data:"):
                    continue
                data = line[5:].strip()
                if data == "[DONE]":
                    break
                chunk = json.loads(data)
                if chunk.get("usage"):
                    print("\n[usage]", chunk["usage"])
                for ch in chunk.get("choices", []):
                    delta = ch.get("delta") or {}
                    if delta.get("content"):
                        parts.append(delta["content"])
                        yield delta["content"]
                    for tc in delta.get("tool_calls") or []:
                        idx = tc.get("index", 0)
                        fn = tc.get("function") or {}
                        if tc.get("id"):
                            tool_acc[idx]["id"] = tc["id"]
                        if fn.get("name"):
                            tool_acc[idx]["name"] += fn["name"]
                        if fn.get("arguments"):
                            tool_acc[idx]["arguments"] += fn["arguments"]
                            # arguments 分片期间不是合法 JSON，此刻 loads 必炸
    stream_chat.last_tools = dict(tool_acc)  # type: ignore[attr-defined]


if __name__ == "__main__":
    for piece in stream_chat([{"role": "user", "content": "用三句话介绍连接池。"}]):
        print(piece, end="", flush=True)
    print("\n工具调用累积结果：", getattr(stream_chat, "last_tools", {}))
```

**今天的动作**

- [ ] 把上面代码写进 `src/llm/stream.py`，跑通打字机效果（注意 `flush=True`，不加就是"憋完再吐"的假流式）。
- [ ] 打印前 5 个原始 chunk 的 JSON，观察 `delta` 的实际形状，存进 `experiments/stream-chunks.md`。
- [ ] 加一个中断实验：打印 3 个 chunk 后 `break`，观察连接是否干净关闭、有没有异常噪声。
- [ ] 故意请求一个会调用工具的问题（没有工具也没关系，手写一个假 tool 定义），验证 `arguments` 是分片到达的。
- [ ] 提交。

**会踩的坑**

1. 每个 chunk 就 `json.loads(accumulate)` 一次 tool 参数——分片期间必然不合法。**先拼完再解析**。
2. 客户端超时按"总时长"设，流式长响应被自己的超时掐断。read timeout 是 chunk 间隔，总时长要单独控制。
3. 忘记流式的 usage 需要显式开关，于是 W21 的评测集拿不到成本数据。
4. 前端用同步 `client.post` 等完整响应，用户看到 15 秒白屏——"能不能流式"决定产品手感，这属于工程约束不是优化项。

**自测**

1. 为什么 `arguments` 不能边收边解析？
2. 流式响应里怎么判断"这次真的成功了"？
3. 打字机效果不生效，你的两个排查点是什么？

---

## 周五 · 多轮状态与历史裁剪

**本质**：Agent 的"记忆"是你在 messages 列表里放什么。放太少模型失忆，放太多爆窗口、变慢、变贵、还会因为无关内容干扰判断。**上下文管理是 Agent 工程的核心技能之一，不是杂活。**

**Java 视角**：像给一个无状态服务设计缓存失效策略。你得决定保留什么、丢弃什么、何时压缩——没有"反正内存够"这个选项，窗口就是硬上限。

**核心讲解**

三种裁剪策略及适用场景：

1. **截尾（truncate）**：保留 system + 最近 N 条。简单、可预测，代价是早期事实丢失（用户第一轮说的名字在第 20 轮消失）。适合工具型短任务。
2. **摘要归档（summarize）**：把旧消息压成一条摘要放进 system 或单独一条 user 消息。省 token 且保留要点，代价是摘要可能失真、且每次摘要本身要调用模型（成本 + 延迟）。适合长对话。
3. **滑动窗口 + 关键事实外挂**：窗口保留最近对话，把结构化事实（用户偏好、已确认参数）单独存到别处（W15 的长期记忆表），每轮注入。这是产品级做法。

**硬规矩**：裁剪不能破坏 `assistant.tool_calls` 与 `tool` 结果的**配对**。把一条 tool 消息的父 assistant 消息裁掉了，服务商直接 400。按 token 硬切的人一周内一定会撞上这条。

**代码**

```python
from src.llm.tokens import estimate_messages


def trim_truncate(messages: list[dict], *, max_tokens: int, keep_system: bool = True) -> list[dict]:
    """保留 system + 最近的成对消息；保证不出现孤立 tool 结果。"""
    head = messages[:1] if keep_system and messages[0]["role"] == "system" else []
    tail = messages[len(head) :]
    out: list[dict] = []
    budget = max_tokens - estimate_messages(head)
    for m in reversed(tail):
        cost = estimate_messages([m])
        if cost > budget:
            break
        # 关键：若裁掉 assistant(tool_calls)，必须连带裁掉它后面的 tool 结果
        if m["role"] == "tool" and (not out or out[0].get("role") != "assistant"):
            continue
        budget -= cost
        out.insert(0, m)
    return head + out


def trim_summarize(messages: list[dict], *, keep_recent: int = 4) -> list[dict]:
    """把早期对话压成一条摘要。用模型压之前先压长度，别把整段原始历史喂进去。"""
    if len(messages) <= keep_recent + 2:
        return messages
    head = messages[:1]
    old, recent = messages[1:-keep_recent], messages[-keep_recent:]
    digest_src = [{"role": m["role"], "content": (m.get("content") or "")[:300]} for m in old]
    from src.llm.lab import chat

    data = chat(
        [
            {"role": "system", "content": "把下面的对话压缩成要点，保留所有事实、数字、用户决定，不要评论。"},
            {"role": "user", "content": str(digest_src)},
        ],
        max_tokens=300,
    )
    summary = data["choices"][0]["message"].get("content") or ""
    return head + [{"role": "user", "content": f"[历史摘要]\n{summary}"}] + recent
```

**今天的动作**

- [ ] 写 `src/llm/history.py`，落地 `trim_truncate`。
- [ ] 造一个 25 轮、含 3 次工具调用的对话历史，用 `trim_truncate` 裁到 2000 token 再发给模型，**确认没有 400**。这是今天唯一的重点。
- [ ] 加断言测试：裁剪结果里不存在"父消息被裁掉的孤立 tool 结果"。
- [ ] 实现 `trim_summarize`，对比同一追问在两种策略下的回答质量，结论写进日志。
- [ ] 提交。

**会踩的坑**

1. 按"消息条数"裁剪但没按 token 裁剪——一条巨大的工具结果就把预算爆了。反之也一样，两者都要有上限。
2. 拆散 `tool_calls` 与 `tool` 配对（已强调，因为一定会踩）。
3. 摘要里丢失了用户 earlier 给的关键约束（比如"只用只读副本"）。摘要提示词里明确写"保留所有事实、数字、用户决定"，并在 W15 用结构化事实表兜住关键约束。

**自测**

1. 为什么截尾策略下第 20 轮会忘记第 1 轮的信息？这在产品上意味着什么？
2. 摘要裁剪本身的成本与延迟怎么估算？什么时候不划算？
3. 孤立 tool 消息为什么会让请求失败？

---

## 周六 · 组装 `llm_lab.py`（120 分钟）

本周收口：把周一到周五的积木拼成一个可交给别人运行的 CLI。

**第一段 55' · 拼装**

- [ ] `uv add typer`（或直接用 `argparse`），建 `scripts/llm_lab.py`：

```python
# scripts/llm_lab.py
from typing import Any

import typer

from src.config import get_settings
from src.llm.history import trim_summarize, trim_truncate
from src.llm.lab import chat
from src.llm.stream import stream_chat
from src.llm.tokens import estimate_messages

app = typer.Typer(help="零框架的多轮对话 CLI")


@app.command()
def chat_cmd(
    prompt: str,
    *,
    model: str = typer.Option(None, help="覆盖默认模型"),
    system: str = typer.Option("你是一个严谨的助手。"),
    stream: bool = typer.Option(True, help="流式输出"),
    max_ctx: int = typer.Option(6000, help="历史 token 预算"),
    history_file: str = typer.Option(".history/default.json", help="会话历史落盘位置"),
) -> None:
    from pathlib import Path

    hist: list[dict[str, Any]] = []
    p = Path(history_file)
    if p.exists():
        import json

        hist = json.loads(p.read_text(encoding="utf-8"))

    messages = [{"role": "system", "content": system}] + hist + [{"role": "user", "content": prompt}]
    # 先估算再裁剪，裁剪后仍超预算就换摘要策略
    if estimate_messages(messages) > max_ctx:
        messages = [messages[0]] + trim_truncate(messages[1:], max_tokens=max_ctx, keep_system=False)
    if stream:
        answer = "".join(stream_chat(messages, model=model or get_settings().llm_model))
        print(answer)
    else:
        data = chat(messages, model=model or get_settings().llm_model)
        answer = data["choices"][0]["message"].get("content") or ""
        print(answer)
        typer.echo(f"[usage] {data.get('usage')}", err=True)

    p.parent.mkdir(parents=True, exist_ok=True)
    hist += [{"role": "user", "content": prompt}, {"role": "assistant", "content": answer}]
    import json

    p.write_text(json.dumps(hist, ensure_ascii=False, indent=2), encoding="utf-8")


@app.command()
def reset(history_file: str = ".history/default.json") -> None:
    """清空会话历史。别小看这条命令，调试时你会每天用十次。"""
    from pathlib import Path

    Path(history_file).unlink(missing_ok=True)


if __name__ == "__main__":
    app()
```

- [ ] 验证三条：`uv run python scripts/llm_lab.py "你好"` 有输出、`--no-stream` 走非流式、连续两次调用能记住上下文、`reset` 后失忆。
- [ ] 把 `.history/` 加进 `.gitignore`（里面有真实对话内容，绝不入库）。

**第二段 55' · 记录与发布**

- [ ] README 增补"零框架对话 CLI"章节：命令示例、三个参数说明、一次真实 usage 截图。
- [ ] 写 `docs/notes/week3-llm-api-本质.md`：面向三个月后的自己，讲清消息结构 / token 计费 / 流式拼接 / 裁剪四件事，附你今天踩到的真实报错。**这篇笔记会被 W23 的技术文复用，值得花 30 分钟写好。**
- [ ] `uv run pytest` + `uv run pyright` 双绿，提交并打 tag：`git tag w03-done`。

**验收标准（自查，别糊弄自己）**

- 能在 60 秒内说出一次 chat completion 请求体的每个字段作用。
- 能徒手写出流式 `tool_calls` 的按 index 拼接逻辑。
- 能回答"第 20 轮对话比第 1 轮贵几倍，为什么"。
- `llm_lab.py` 在一台干净机器上，照 README 三条命令能跑起来。

---

## 本周术语卡

| 术语 | 一句话解释 | 为什么重要 |
|---|---|---|
| messages | 每次请求发送的完整历史列表 | 模型无状态，这是唯一的记忆来源 |
| delta | 流式响应的增量片段 | 必须累加，任何单个 chunk 都不完整 |
| `finish_reason` | 结束原因：`stop`/`length`/`tool_calls` | 判断是否被截断、是否要走工具分支 |
| token | 模型处理与计费的最小单位 | 窗口、成本、延迟三者都由它决定 |
| 上下文窗口 | 输入 + 输出（+ 思考）token 的上限 | Agent 一切"塞不下"问题的根源 |
| prompt caching | 固定前缀命中缓存按折扣计费 | 最易获取的成本收益，W07 实测 |
| 截断裁剪 | 保留 system + 最近若干条 | 简单但会丢早期事实 |
| 摘要裁剪 | 旧对话压成一条摘要 | 省 token，代价是失真与额外调用 |
| 配对完整性 | `assistant.tool_calls` 与其 `tool` 结果必须同生共死 | 裁错了直接 400 |
| 概率约束 | 提示词影响倾向而非保证 | 决定了为什么必须做 schema 校验与重试 |

## 延伸阅读（通勤时看）

- 你所用服务商的 Chat Completions 参考页——把这周用到的字段全部过一遍，尤其 `stream_options`、`tool_choice`、`parallel_tool_calls` 三个（后两个 W06 用）。
- OpenAI Cookbook 的 "How to count tokens" 一节，理解各家 tokenizer 差异。
