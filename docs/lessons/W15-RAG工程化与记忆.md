# W15 · RAG 工程化与记忆：从"能查准"到"能上线"

> 本周在计划中的位置：阶段 3 收口周。W14 让检索有了数字，本周让检索**可信**（有出处、会拒答、守边界）并让 Agent **有记忆**（短期会话、长期事实、语义三类分开存），周六把它整体接进项目一，交付项目一 v2。

## 本周导航

| 日 | 主题 | 今天的产出物 |
|---|---|---|
| 周一 | 引用与可追溯 | `citations.py`：答案带片段 id、终端可展开原文 |
| 周二 | 拒答与置信度阈值定标 | 阈值 + 5 条负样本验证表 |
| 周三 | Agentic RAG：检索做成工具 | `tools/rag_search.py` 接进 W06 工具循环 |
| 周四 | 记忆三层：会话 / 事实 / 语义 | 三层读写设计 + `memory/` 最小实现 |
| 周五 | 索引维护：增量、去重、版本 | `reindex.py --since` |
| 周六 | 全部接入项目一，端到端演示 | 项目一 v2 + README 更新 |
| 周日 | 休息，不许补课 | — |

本周的判断标准从"查得准"换成"敢给别人用"。引用、拒答、权限这三个东西缺任何一个，你的项目在评审里都会被归为 demo。

---

## 周一 · 引用与可追溯：让答案指回原文

**本质**

引用（citation）机制做三件事：检索返回的每个片段带稳定 id；prompt 里把 id 与内容一起给模型，并要求它标注依据；输出后把标记解析出来、映射回原文供人点开。它同时压幻觉和降核查成本——模型**只能引用你给它的 id**，编造就当场可判；而审阅者不用重跑检索，点一下就能看到证据。

**Java 视角**

等于给接口返回体加 `evidenceRefs`，并在网关侧做一次 `assert refs ⊆ 本次查询实际返回的行`。引用校验是契约校验的一种，和 W12 补练 C 的"引用行号必须真实存在"是同一个模式，只是主键从行号换成了 chunk_id。

**核心讲解**

引用降低幻觉的机制不是玄学，而是三条具体约束：

1. **约束解码空间**：要求"每个结论句后必须跟 `[c:<chunk_id>]`"，模型必须先找到一段支持它的文本。没有依据的句子要么被省略、要么标记缺失，可被程序检出。
2. **可判定**：解析出的 id 集合必须是本轮实际送入片段的子集。任何未出现的 id = 编造，直接拒绝输出并重问。这是能自动化的红线，比"提示词让它诚实"强得多。
3. **可核查**：人工抽检从"整段答案对不对"变成"这条引用的原文支不支持这句话"，一分钟能查 10 条。真实上线后的日常质控只有这一种形式跑得动。

工程形态与坑：

- **id 用短标签而不是 16 位哈希**。给模型 `[1] [2] [3]`，服务端维护 `1 -> chunk_id` 映射。理由：长哈希会被模型截断、改写、粘连，且白占 token。映射表随响应一起返回，前端按序号展开原文。
- **片段必须带来源信息**（`doc_id` + `heading` + 前 80 字），只给 id 的引用没有可读性，人不会去查。
- **模型会"过度引用"**：整段答案挂一个引用，或给没有依据的推论也挂上。缓解：提示里要求"每个事实性句子各挂自己的引用；无法对应的句子不要挂"。别指望完全消除，人工抽检是必须的后手。
- **引用不覆盖多跳合成**："A 部门上限比 B 高 3 万行"这种结论是两段相减得出的，模型只能各引一段。这属于正常，不是 bug——但你的校验器要允许多引用（`[1][3]`），别把它当格式错误拦下。

**代码**

```python
"""citations.py — 短标签 + 双向映射 + 引用真实性校验。"""
from __future__ import annotations

import re
from dataclasses import dataclass, field

CITE_RE = re.compile(r"\[(\d{1,2})\]")


@dataclass(slots=True)
class Evidence:
    idx: int                 # 给模型看的短标签
    chunk_id: str
    doc_id: str
    heading: str
    content: str


@dataclass(slots=True)
class Answer:
    text: str
    citations: list[int] = field(default_factory=list)
    fabricated: list[int] = field(default_factory=list)     # 模型引用了不存在的编号


def build_prompt(question: str, evidences: list[Evidence], style: str = "concise") -> str:
    """把证据编号化后注入。style 之类的非检索参数别塞进 prompt，那是 W19 的事。"""
    block = "\n\n".join(
        f"[{e.idx}] 来源={e.doc_id}｜小节={e.heading or '-'}\n{e.content}" for e in evidences)
    return (
        "只依据下面编号资料回答。每个事实性句子后面紧跟它对应的编号标记，形如 [1]；"
        "多个依据写作 [1][3]。资料里没有的，直接说'资料中未说明'，不要用自己的知识补齐，"
        "也不要引用任何未给出的编号。\n\n--- 资料 ---\n" + block +
        f"\n--- 资料结束 ---\n\n问题：{question}\n")


def parse_answer(raw: str, evidences: list[Evidence]) -> Answer:
    valid = {e.idx for e in evidences}
    cited = [int(m) for m in CITE_RE.findall(raw)]
    # 顺序去重：dict.fromkeys 保留首次出现顺序，前端按引用出现次序展示
    return Answer(text=raw, citations=list(dict.fromkeys(cited)),
                  fabricated=[i for i in dict.fromkeys(cited) if i not in valid])


def render(answer: Answer, evidences: list[Evidence]) -> str:
    """终端版展开；前端把同一份 evidences 交给点击事件即可，数据结构不变。"""
    by_idx = {e.idx: e for e in evidences}
    out = [answer.text]
    if answer.fabricated:
        out.append(f"\n!! 引用异常：{answer.fabricated} 不在本轮证据内，本条回答不可信")
    out.append("\n—— 引用原文 ——")
    for i in answer.citations:
        e = by_idx.get(i)
        if e:
            out.append(f"[{i}] {e.doc_id} / {e.heading or '-'} (chunk={e.chunk_id})\n    {e.content[:160]}")
    return "\n".join(out)


def ask(question: str, evidences: list[Evidence], call_llm) -> Answer:
    """call_llm(prompt: str) -> str，用你 W03/W05 那个客户端即可。"""
    ans = parse_answer(call_llm(build_prompt(question, evidences)), evidences)
    if ans.fabricated:                       # 编造引用 = 拒绝输出，重问一次；仍编造则降级为"无依据"
        ans = parse_answer(call_llm(build_prompt(question, evidences) +
                                    f"\n\n注意：上一次你引用了不存在的编号 {ans.fabricated}。"), evidences)
        if ans.fabricated:
            ans = Answer(text="（引用校验未通过，已降级）本轮无法给出有依据的回答。")
    return ans
```

**今天的动作**

- [ ] 把 W14 的检索结果包装成 `Evidence`（编号从 1 开始，`chunk_id` 直接来自库里那条记录）。
- [ ] 跑 5 条标注集问题，人工核对每条引用：原文是否真支持那句话？过度引用有几个？
- [ ] 用"故意只给 2 段不相关资料"制造一次编造引用，确认 `fabricated` 捕获 + 重问 + 降级三步都走到。
- [ ] 打印一次 `render()` 的终端输出，确认你能在 10 秒内定位到某条引用的出处（做不到就是格式设计有问题）。
- [ ] commit，并在 README 里加一段"如何核查一条回答"。

**会踩的坑**

1. **把 16 位 chunk_id 直接丢给模型当标签**：模型写成 `[a3f1c9]` 或截断，解析正则全废。用序号映射。
2. **引用校验漏在流式路径**：终端流式输出时你边渲染边收，`fabricated` 检测只在结束后做，用户已经看到编造内容。当前阶段先不用流式做引用；要用的话改成"引用标记出现即校验"。
3. **一份证据给多个问题复用**：`evidences` 与本轮调用必须一一对应，缓存复用会让编号指向不同片段，引用集体错位。诊断：trace 里把 `idx -> chunk_id` 映射一起落盘。
4. **只存最终文本没存证据集**：第二天想复现"这条回答当时看到的是什么"，做不到。今天开始 trace 里存 `evidences` 全量（注意 W14 周五说的密级脱敏）。

**自测**

1. 引用机制为什么能"自动检出编造"，而"别编"这类提示词不行？说出可判定的那个不变式。
2. 短标签 + 服务端映射，而不是直接用 chunk_id 当标签，两个理由是什么？
3. 多跳结论的两段引用是正确行为，你的校验器要怎样设计才不会把它误杀？

---

## 周二 · 拒答与置信度阈值：给"不知道"标定一条线

**本质**

库里没有答案时，检索仍会返回 top-5（ANN 永远返回最近的 5 个，哪怕很远）。这是 RAG 最危险的结构缺陷：模型拿到一堆不相关资料，会尽力从中编出一个答案。拒答机制 = 在分数低于阈值时**根本不进生成**，明确输出"资料中没有"。阈值不是拍脑袋，是从标注集的正例与负例分数分布里选出来的。

**Java 视角**

等同于给"查不到"设计正确的接口契约：返回空集合而不是返回默认值。区别在于 RAG 里"空"要靠你自己判——最近邻永远存在，`isEmpty` 恒为 false，只能用距离阈值造一个语义上的空。

**核心讲解**

标定流程（今天的正事）：

1. **正例分数**：W14 那 20 条 positive，取每条 top-1 的分数（用哪一路就用哪一路的分——`cosine` 或 `rerank_score`，两者尺度不可混用，W14 周三坑 #1）。
2. **负例分数**：造 5~8 条"库里确实没有"的问题。造法有讲究，要**贴着库的语义边界造**才有区分度："你们食堂周五供应什么菜"（库是运维手册）、"iOS 19 的新特性"、"这个项目的 CEO 是谁"、把库里的真实条款改一个不存在的参数（库里讲"导出上限 5 万行"，就问"导出上限 3 万行时怎么申请特批"——注意这种"近似但无解"的负例最容易骗过模型，也最能测出阈值）。
3. **看分布**：把 25 个分数排序，找正例下界与负例上界之间的空隙。经验上常见的是正例 top-1 cosine 0.55~0.85、负例 0.30~0.50，重叠 0.50~0.60 一带是灰区。
4. **选阈值**：你要的业务决定取舍。面向"不许瞎说"的场景（合规、审批、内部制度），阈值往高取，宁可多拒答（false reject）；面向"别老说不知道"的场景（助手、客服），往低取，宁可承认偶尔编。今天先按前者取，把选择的理由写进配置注释。
5. **验证**：5 条负例必须 5/5 拒答；20 条 positive 的拒答率（即误拒）要报出来。两个数字一起写进表格，只报一个没意义。

诚实的技术边界：**余弦相似度不是校准过的概率**。同一个 0.62，在换了 embedding 模型、改了 chunk_size、甚至语料涨了一倍之后含义就变了。所以：阈值必须跟着索引版本一起重标定，写进配置并在 `reindex.py`（周五）里打印"当前阈值对应旧索引"的提醒。更稳的做法是相对判据（top-1 与 top-5 的分差、或"top-1 分数 < 该查询 BM25 命中数"），但今天先做绝对阈值 + 每季重标定，简单且可解释。

**代码**

```python
"""gate.py — 阈值标定与拒答判定。"""
from __future__ import annotations

import json
import statistics
from pathlib import Path
from pydantic import BaseModel, Field


class Threshold(BaseModel):
    metric: str = "cosine"          # cosine / rrf / rerank_score —— 尺度不同，禁止跨用
    min_top1: float = Field(gt=0, lt=1)
    reason: str = ""                # 为什么取这个值，三个月后你自己要看
    index_fingerprint: str = ""     # 与索引绑定：换 embedding / 改切分后必须重标定


def top_scores(queries: list[str], retrieve) -> list[tuple[str, float]]:
    """retrieve(q) -> list[(chunk_id, score)] 降序；只要 top-1。"""
    return [(q, retrieve(q)[0][1] if retrieve(q) else 0.0) for q in queries]


def choose_threshold(pos: list[float], neg: list[float]) -> dict:
    """朴素判据：取'正例 10 分位'与'负例上界'的中点；分布重叠时会给出警告而不是假装能分。"""
    pos_lo = sorted(pos)[max(0, int(len(pos) * 0.10))]
    neg_hi = max(neg)
    overlap = neg_hi >= pos_lo
    mid = (pos_lo + neg_hi) / 2
    return {"pos_p10": round(pos_lo, 4), "pos_median": round(statistics.median(pos), 4),
            "neg_max": round(neg_hi, 4), "neg_median": round(statistics.median(neg), 4),
            "suggested": round(mid, 4), "overlap": overlap,
            "note": "重叠：绝对阈值无法干净分开，考虑改用 rerank 分数或相对判据"}


def gate(query: str, hits: list[tuple[str, float]], th: Threshold) -> tuple[bool, str, float]:
    """返回 (是否放行, 状态, top1 分数)。状态区分 no_result / blocked，前端文案不同但都不进生成。"""
    if not hits:
        return False, "no_result", 0.0
    top1 = hits[0][1]
    if top1 < th.min_top1:
        return False, "below_threshold", top1
    return True, "ok", top1


if __name__ == "__main__":
    import os
    from emb_probe import embed, normalize
    from hybrid import vector_search
    from metrics import load_labels

    dsn = os.environ["PG_DSN"]
    def retrieve(q: str) -> list[tuple[str, float]]:
        qv = normalize(embed([q])[0])
        return vector_search(dsn, qv, 5)

    positives = [l.question for l in load_labels()]
    negatives = [
        "食堂周五供应什么菜",
        "iOS 19 的新特性有哪些",
        "导出上限 3 万行时怎么申请特批",      # 库里只有 5 万行，这是"近似但无解"的硬负例
        "这个系统 CEO 是谁",
        "帮我预测明年三季度的工单量",
    ]
    pos = [s for _, s in top_scores(positives, retrieve)]
    neg = [s for _, s in top_scores(negatives, retrieve)]
    print(json.dumps(choose_threshold(pos, neg), ensure_ascii=False, indent=2))
    Path("experiments/w15_scores.jsonl").write_text(
        "\n".join(json.dumps({"q": q, "top1": round(s, 4), "kind": k}, ensure_ascii=False)
                  for k, arr in (("pos", positives), ("neg", negatives))
                  for q, s in top_scores(arr, retrieve)) + "\n", encoding="utf-8")
```

**今天的动作**

- [ ] 造 5~8 条负样本，其中至少 2 条"近似但无解"型（改参数、改条件），存进 `eval/labels.jsonl`（`kind: "negative"`）。
- [ ] 跑 `__main__`，把 25 个分数排序打印出来，**肉眼找空隙**，不要只看 `suggested`。
- [ ] 定阈值并把 `metric/reason/index_fingerprint` 写进 `config/threshold.json`。
- [ ] 验证：5 条负样本 5/5 拒答；20 条正样本误拒率报出数字。任一不达标就调阈值或加 rerank 分数作第二判据。
- [ ] 端到端跑一次"负样本 + 生成"（不设阈值），把模型编出来的答案贴进笔记，再设阈值跑一次对比。这两段输出并排贴出来，就是 W23 技术文里最有说服力的一组图。

**会踩的坑**

1. **阈值用 BM25 原始分数**：BM25 无上界且随语料漂移，0.5 这条线毫无意义。诊断：`gate` 对同一条查询在不同语料版本下给出不同判定。
2. **拒答文案泄露信息**：`blocked` 时说"资料中存在但你无权查看"就是泄漏（W14 周五提过）。统一输出"资料中未说明"。
3. **只有 top-1 判据，漏掉"全部不相关但 top-1 偶然高"**：一条宽泛的总则性 chunk（比如"本制度适用于全体员工"）与什么问题都挺像，长期霸占 top-1 并把闸门顶开。诊断：把 5 条负样本的 top-1 chunk_id 打出来，如果都指向同一段，就是它。修法：给这类"万能段"加 `metadata.is_boilerplate` 并在检索层排除，或要求"top-1 与 top-3 都要过阈值"。
4. **误拒率不报**：只演示了"能拒答"，没证明"还能正常答题"。这条在评审时会被追问。

**自测**

1. 为什么 RAG 里"检索返回空"几乎不可能发生，因此必须人为造阈值？
2. 换 embedding 模型后为什么必须重标定阈值？用今天 `index_fingerprint` 字段说明你的处置流程。
3. "近似但无解"的负样本为什么比"食堂菜谱"这种无关负样本更有价值？

---

## 周三 · Agentic RAG：把检索交给模型决定

**本质**

固定检索是"每个问题都先查一次"。Agentic RAG 是把检索注册成工具，模型自己决定要不要查、查什么词、查几次、要不要换个词再查。收益来自**它能观察到第一次检索的结果并修正查询**；代价是每次决策都多一轮模型调用。

**Java 视角**

固定检索是 Service 里硬编码的一次 DAO 调用；Agentic RAG 是把这个 DAO 方法暴露成 RPC 接口，由编排层（模型）决定调用次数与参数。你在 W06 手写的工具循环就是那个编排层——今天只是往 `TOOL_REGISTRY` 里加一项。

**核心讲解**

三条路径的对比要量化，别用体感：

| 维度 | 固定检索 | Agentic RAG |
|---|---|---|
| 简单事实题延迟 | 1 次 embedding + ANN + 1 次生成（最优） | 至少多 1 轮决策，慢 1~2 倍 |
| 无需检索的寒暄/推理题 | 照样查，白烧 5 段 token | 模型可跳过，省一次检索与注入 |
| 多跳题 | 一次查询只能覆盖一跳，recall 受限 | 可"查到 A 里提到 B，再查 B"，明显更强 |
| 问法不匹配 | 一次机会 | 可自我改写重试（等价于内联 multi-query） |
| 成本可控性 | 确定（每问 1 次检索） | 不确定，必须设 `max_tool_calls` |
| 可评测性 | 指标稳定 | 同一题不同跑法结果会变，评测要多跑取均值 |

设计要点：

- **工具返回要"可推理"而不是"可展示"**：把 `chunk_id/heading/preview/score` 结构化返回，模型才能判断"够不够、要不要再查"。只返回拼接好的长文本，模型失去判断依据，会变成"再查一次保险"。
- **给模型一条退出路径**：检索 2 次仍不满意时应当回答"资料不足"（接昨天的闸门），而不是无限重试。没有出口就是烧钱死循环——这条和你 W06 的最大步数熔断是同一个机制，只是这次预算是"检索次数"。
- **工具描述（description）就是路由提示**：写清"什么时候该用/不该用"，比在系统提示里讲一堆有效。这是 Function Calling 里最被低估的一行字。
- **别把检索做成唯一工具**：如果库里明确有答案的题型占多数，混合策略更实用——先跑固定检索，分数落在灰区（比如阈值 ±0.05）才升级为 agentic 模式。这是线上常见形态，也值得你在笔记里记一句"我的灰区比例是多少"。

**代码**

```python
"""tools/rag_search.py — 检索作为工具，复用 W06 的工具循环，不改循环本身。"""
from __future__ import annotations

import json
import os
from typing import Any

from citations import Evidence
from gate import Threshold, gate
from hybrid import Bm25Index, load_corpus, rrf, vector_search
from emb_probe import embed, normalize

TOOL_SPEC: dict[str, Any] = {                     # OpenAI 兼容 function schema（字段名按你服务商核对）
    "type": "function",
    "function": {
        "name": "rag_search",
        "description": "在内部文档库中检索资料。需要事实依据（制度、参数、流程、报错含义）时使用；"
                       "纯改写/计算/闲聊不要调用。若首次结果不足，可用不同关键词再调用一次，最多两次。",
        "parameters": {
            "type": "object",
            "properties": {
                "query": {"type": "string", "description": "检索语句，尽量带领域名词"},
                "top_k": {"type": "integer", "default": 5, "maximum": 10},
            },
            "required": ["query"],
        },
    },
}


class RagSearch:
    def __init__(self, dsn: str, th: Threshold) -> None:
        self.dsn = dsn
        self.th = th
        self.bm = Bm25Index(load_corpus(dsn))

    def __call__(self, args: dict[str, Any]) -> str:
        """返回信封 JSON：模型据此决定下一步，所以 status 与 reason 必须写清。"""
        q = str(args.get("query", "")).strip()
        k = min(int(args.get("top_k", 5)), 10)
        if not q:
            return json.dumps({"ok": False, "error": "query 不能为空"}, ensure_ascii=False)
        qv = normalize(embed([q])[0])
        fused = rrf([self.bm.search(q, 30), vector_search(self.dsn, qv, 30)])[:k]
        ok, status, top1 = gate(q, fused, self.th)
        hits: list[dict[str, Any]] = []
        for i, (cid, score) in enumerate(fused, 1):
            row = self.fetch(cid)
            if row:
                hits.append(Evidence(idx=i, chunk_id=cid, doc_id=row["doc_id"],
                                     heading=row["heading"], content=row["content"]).__dict__)
        return json.dumps({
            "ok": True, "status": status if ok else "refuse", "top1_score": round(top1, 4),
            "hint": ("资料充分，可作答并引用 [n]" if ok else
                     "检索得分低于阈值：请回答'资料中未说明'，不要重试超过一次，不要编造"),
            "evidences": hits,
        }, ensure_ascii=False)

    def fetch(self, chunk_id: str) -> dict[str, Any] | None:
        import psycopg
        with psycopg.connect(self.dsn) as conn:
            r = conn.execute("SELECT doc_id, heading, content FROM chunks WHERE chunk_id=%s", (chunk_id,)).fetchone()
        return {"chunk_id": chunk_id, "doc_id": r[0], "heading": r[1], "content": r[2]} if r else None
```

接进循环只需一行注册：`TOOL_REGISTRY["rag_search"] = RagSearch(dsn, th)`，schema 列表加上 `TOOL_SPEC`。W06 的循环逻辑不动——这是它该有的样子：工具是插件，不是分支。

**今天的动作**

- [ ] 注册 `rag_search`，跑 3 条问题：一条无需检索（"帮我把这句话改正式点"）、一条单跳、一条多跳，把模型的 tool_calls 序列打印出来。
- [ ] 确认它确实**跳过**了第一条（没跳过就改 description，这比调温度有用）。
- [ ] 加 `max_search_calls = 2` 熔断，故意给一条库里没有的题，观察它是否重试到上限并走拒答。
- [ ] 同一批 20 条标注题分别跑固定检索与 agentic，记三列：平均 LLM 调用次数、平均延迟、平均输入 token。指标（hit-rate）用 W14 的 `evaluate`。
- [ ] 结论写进笔记："我的场景 agentic 值不值"，给出那三列数字。

**会踩的坑**

1. **没有次数上限**：模型反复检索同一道题，一晚烧掉半周预算。诊断：`learning-log` 旁边贴用量统计；trace 里 tool_call 计数。这条必须今天堵。
2. **工具返回塞满全文**：一次返回 10 段全文，模型不再检索第二跳（它以为信息够了），token 还爆。返回 preview + 结构，需要全文用 `chunk_id` 再取。
3. **检索词由模型改写后丢掉编号**：同 W14 周四坑 #1，模型把 `SO-2026-1183` 改写成"某订单"。修法：description 里明写不许改写编号，或在工具内做"原词 + 改写词"两路 RRF。
4. **固定检索与 agentic 混着评测**：两条路径指标不可比，本周表格就废了。评测时统一同一套 prompt 与模型版本。

**自测**

1. Agentic RAG 相比固定检索的真实收益来源是哪一条机制（不是"更智能"）？
2. 为什么工具返回值里的 `status`/`hint` 比返回更多文本更能提升多跳表现？
3. 你的项目里什么条件下选"固定检索 + 灰区升级 agentic"而不是纯 agentic？

---

## 周四 · 记忆三层：会话、事实、语义

**本质**

记忆不是一个东西。三类问题的存储与召回方式完全不同：**这段对话刚说了什么**（短期，时序）、**关于这个用户/任务确定成立的事实**（长期事实，结构化）、**用户历史交互里可复用的知识**（语义，向量）。把三类混成一个"记忆"，你就只能用摘要解决所有问题——而摘要会失真，关键约束失真是事故。

**Java 视角**

正是三级缓存的分层：L1 请求上下文（会话 state）、L2 数据库主数据（`user_memory` 表，有 schema、有约束）、L3 全文/向量检索。区别在于 L2 的写入方是模型，所以要当不可信输入校验——`model_validate` + 枚举 + 上限，跟你校验任何外部入参一样。

**核心讲解**

| 层 | 存什么 | 写入时机 | 召回方式 | 淘汰策略 |
|---|---|---|---|---|
| 短期会话 | 完整消息序列、工具结果 | 每轮节点边界自动快照（checkpointer） | 同 `thread_id` 自动带回 | 会话 TTL；过长时截断（保系统提示 + 最近 N 轮 + 关键事实卡） |
| 长期事实 | 结构化：偏好、已确认参数、约束、环境名 | 显式事件驱动：用户明确声明、校验通过、任务确认完成时 | 每轮开始按 `user_id` 全量读，注入系统提示 | 按 key 覆盖写（upsert）；带 `updated_at`；不常用项定期归档，别删 |
| 语义记忆 | 历史对话摘要、常见问题、踩过的坑 | 任务结束批处理写 | 向量检索，按需拼进上下文 | 时间衰减 + 命中计数，低分归档 |

**必须记住的那条约束**：像"该用户的服务只能查预发环境""导出上限一律按 3 万审批"这类**关键约束不能只靠会话摘要传递**。摘要是模型生成的、有损的、且随轮次滚动重写；第 20 轮时它可能已经被摘要掉或改写走形，而模型会若无其事地继续跑。正确做法：落到 `user_memory` 的固定字段（`env_scope = 'staging'`），每轮开始由代码读出来、硬拼进系统提示，并让校验器检查违规动作。摘要只做辅助叙述，不做约束载体。

技术形态（沿用 W10/W12 已核实的 API）：`pip install langgraph-checkpoint-sqlite`，`from langgraph.checkpoint.sqlite import SqliteSaver`，`with SqliteSaver.from_conn_string("checkpoints.sqlite") as saver: builder.compile(checkpointer=saver)`，调用 `app.invoke(..., {"configurable": {"thread_id": f"user:{uid}:chat:{sid}"}})`；读快照 `app.get_state(cfg)`。两条军规不变：接了 checkpointer 后每轮只传增量消息；`thread_id` 来自业务会话号。

**代码**

```python
"""memory/store.py — 三层里的 L2：结构化长期事实。校验后再写，读取即注入。"""
from __future__ import annotations

import json
import os
from typing import Any, Literal

import psycopg
from pydantic import BaseModel, Field, field_validator

KEYS = {"env_scope", "default_service", "approval_limit", "tone", "forbidden_action"}
Fact = Literal["user_stated", "tool_verified", "inferred"]        # 来源决定可信度与覆盖规则


class MemoryItem(BaseModel):
    user_id: str
    key: str
    value: str
    source: Fact = "user_stated"
    confidence: float = Field(default=1.0, ge=0, le=1)

    @field_validator("key")
    @classmethod
    def _key(cls, v: str) -> str:
        if v not in KEYS:
            raise ValueError(f"未知记忆键 {v}；新增键要先想清楚它的覆盖与淘汰规则")
        return v

    @field_validator("value")
    @classmethod
    def _len(cls, v: str) -> str:
        v = v.strip()
        if not v or len(v) > 200:
            raise ValueError("记忆值必须非空且 <=200 字：长篇内容属于语义记忆，不该进这张表")
        return v


UPSET = """
    INSERT INTO user_memory (user_id, key, value, source, confidence, updated_at)
    VALUES (%(u)s, %(k)s, %(v)s, %(s)s, %(c)s, now())
    ON CONFLICT (user_id, key) DO UPDATE
    SET value = EXCLUDED.value, source = EXCLUDED.source,
        confidence = EXCLUDED.confidence, updated_at = now();
"""
# 只有 user_stated / tool_verified 允许覆盖旧值；inferred 不得覆盖已确认事实（防止模型臆测污染）
GUARD = {"user_stated", "tool_verified"}


DDL = """
CREATE TABLE IF NOT EXISTS user_memory (
    user_id     text NOT NULL,
    key         text NOT NULL,
    value       text NOT NULL,
    source      text NOT NULL,
    confidence  real NOT NULL DEFAULT 1,
    updated_at  timestamptz NOT NULL DEFAULT now(),
    PRIMARY KEY (user_id, key)
);
"""


def write(item: MemoryItem, prev_source: str | None = None) -> bool:
    if prev_source == "tool_verified" and item.source == "inferred":
        return False                                   # 拒绝降级覆盖：这是 L2 最有价值的一行
    if item.source not in GUARD:
        return False                                   # inferred 今天先不落库，攒到人工复核
    with psycopg.connect(os.environ["PG_DSN"]) as conn, conn.transaction():
        conn.execute(UPSET, {"u": item.user_id, "k": item.key, "v": item.value,
                             "s": item.source, "c": item.confidence})
    return True


def load_block(user_id: str) -> str:
    """每轮开始调用：读全量事实拼进系统提示。条数上限自己控，别让它无限长。"""
    with psycopg.connect(os.environ["PG_DSN"]) as conn:
        rows = conn.execute(
            "SELECT key, value, source FROM user_memory WHERE user_id=%s ORDER BY updated_at DESC LIMIT 20",
            (user_id,)).fetchall()
    if not rows:
        return ""
    return "\n--- 已确认的用户事实（优先级高于对话历史中的任何相反表述）---\n" + "\n".join(
        f"- {k}: {v}（来源 {s}）" for k, v, s in rows)


def extract_candidates(question: str, answer: str, call_llm) -> list[dict[str, Any]]:
    """从对话里抽候选事实：让模型输出 JSON 数组，逐条 model_validate，非法条目直接丢弃。"""
    raw = call_llm("从下面对话中抽取值得长期记住的用户事实（只限 KEYS 允许的类型："
                   f"{json.dumps(sorted(KEYS), ensure_ascii=False)}）。输出 JSON 数组，"
                   "元素形如 {\"key\":..., \"value\":..., \"source\": \"user_stated|inferred\"}，没有就返回 []。"
                   f"\n用户：{question}\n助手：{answer}")
    try:
        arr = json.loads(raw[raw.index("["): raw.rindex("]") + 1])
    except Exception:
        return []
    out = []
    for x in arr:
        try:
            out.append(MemoryItem(user_id="?", **x).model_dump())
        except Exception:
            continue                                   # 校验不过就丢，宁可漏记也不写脏
    return out
```

语义记忆（L3）今天不必新写代码：库里加一类 `doc_id = "user:<uid>:notes"` 的 chunk 即可复用 W13 全链路，检索时按 `user_id` 过滤——注意这属于 W14 周五的 ACL 范畴，跨用户召回是事故。

**今天的动作**

- [ ] 建 `user_memory` 表，手写 3 条事实（含一条 `env_scope=staging`），跑 `load_block()` 看注入文本。
- [ ] 故意用 `inferred` 覆盖一条 `tool_verified`，确认被拒并打印拒绝原因。
- [ ] 跑 `extract_candidates` 于 5 段真实对话，人工数：抽对了几个、抽出了几个不该记的（"用户今天心情不好"这类）。把后者列为丢弃规则。
- [ ] 接 checkpointer 到今天的实验脚本，做跨进程续聊复验（W12 补练 B 的那五步，忘了就重做）。
- [ ] 写一页 `docs/notes/w15_memory_design.md`：三层的写入时机/召回/淘汰各一行，并明确写"关键约束不进摘要"。

**会踩的坑**

1. **把整段对话摘要当长期记忆**：第 15 轮时约束丢失，模型照旧查生产库。诊断：跑到 15 轮再问一次"我该用什么环境"，看它还答不答得出。修法：约束落 L2 字段。
2. **记忆无限增长**：`load_block` 注进 200 条，系统提示比证据段还长，token 成本与注意力都被挤走。淘汰策略要今天定，不是以后。
3. **模型臆测写入事实表**：用户说"这次先在预发跑"，模型记成 `env_scope=staging`（永久）。修法：`source=inferred` 不落库；只有明确声明或工具验证过的才写；值里带"仅本次"的一律不记。
4. **跨用户串味**：语义记忆检索没带 `user_id` 过滤，A 的历史进了 B 的上下文。就是 W14 周五那条红线在记忆层的复现，今天的红队测试要包含一条跨用户断言。

**自测**

1. 为什么 checkpointer 存的不算长期记忆？两者在"生命周期"上的关键差别是什么？
2. `confidence` 与 `source` 两个字段各自防的是什么问题？
3. 举一个"摘要失真导致事故"的具体场景，说出你的结构化替代方案。

---

## 周五 · 索引维护：增量、去重、版本

**本质**

真实文档会变。索引维护要解决：只重建变了的文档、不产生重复副本、旧版本不残留在检索路径里、重建失败不把整个库搞坏。核心手法是**软删旧版本 + 新版本全部写入 + 最后原子切换 active 标记**，而不是 W13 那种"先 DELETE 再 INSERT"。

**Java 视角**

就是蓝绿发布搬到数据层：新旧两张版本并存，读侧靠一个 `WHERE active = true` 切换，切换是一个 UPDATE（或直接换视图），失败可回滚。W13 的先删后插相当于停服部署——能跑通 demo，不能上线。

**核心讲解**

四个动作，逐个说清为什么：

1. **变更检测**：以文件内容哈希为指纹（不是 mtime——同步盘、git checkout、scp 都会改 mtime 而内容不变，会造成大量无谓重嵌入；mtime 只适合做**预筛**：先比 mtime 再比哈希，两层）。`reindex.py --since 7d` 的语义是"mtime 比这晚的候选文件里，再挑内容哈希与库里不同的"。
2. **chunk 级去重**：同一文档内重复段落（表格表头、条款复读）用 `chunk_id`（内容哈希）主键天然合并；跨文档重复（列表页与详情页同一段）用 `content_hash` 唯一约束 + 保留 `doc_id` 优先级最高那份。注意：去重会让 `relevant_ids` 少一个 id，标注集要同步（W14 周一坑 #1 的兄弟问题）。
3. **版本标记与软删**：表上加 `doc_hash`、`active boolean`、`indexed_at`。重建文档 D 时：写入 D 的新 chunk（`active=true`），把 D 的旧 chunk `UPDATE ... SET active=false`（保留可追溯与回滚），检索 SQL 一律加 `AND active = true`。旧行不真删——它的价值是"出问题能查上周那版为什么命中"。真删交给定期 `DELETE WHERE active=false AND indexed_at < now()-'30 days'`。
4. **切换与一致性**：新 chunk 嵌入 + 入库放**一个事务**里，且切换 `active` 是事务的最后一条语句；中途失败则整个文档保持旧版本可检索，不会出现"旧的下线了、新的没进来"的空洞。`--dry-run` 先打印影响面（几个文档、几条 chunk、预计 token），是你每次真跑前的固定动作。

还有一件本周最容易漏的：**任何索引变更后必须重跑 `metrics.py`**。切分参数没变、内容只改了两段，指标也可能掉——W14 周六固定变量的对立面就是"运维期没有可比性"。所以 `reindex.py` 结束时打印"重跑评测：python metrics.py --gate"，把习惯钉在脚本里。

**代码**

```python
"""reindex.py — 增量重建：mtime 预筛 -> 哈希确认 -> 新行入库 -> 软删旧行 -> 原子切换。"""
from __future__ import annotations

import argparse
import hashlib
import os
import time
from datetime import datetime, timedelta
from pathlib import Path

import psycopg

from build_index import load_chunks                 # 复用 W13 的解析+切分入口
from emb_probe import embed, normalize


def sha(text: str) -> str:
    return hashlib.sha256(text.encode()).hexdigest()[:16]


def candidates(root: Path, since_days: int | None) -> list[Path]:
    if since_days is None:
        return sorted(p for p in root.glob("*") if p.suffix in {".md", ".txt", ".html"})
    cutoff = datetime.now() - timedelta(days=since_days)
    return [p for p in root.glob("*") if p.suffix in {".md", ".txt", ".html"}
            and datetime.fromtimestamp(p.stat().st_mtime) >= cutoff]      # mtime 只用于预筛


def changed(conn, docs: list[Path]) -> list[Path]:
    """第二层判据是内容哈希：mtime 变而内容未变的文件全部跳过，省掉整批 embedding。"""
    out = []
    for p in docs:
        new_hash = sha(p.read_text(encoding="utf-8", errors="replace"))
        row = conn.execute("SELECT DISTINCT doc_hash FROM chunks WHERE doc_id=%s AND active", (p.stem,)).fetchone()
        if row is None or row[0] != new_hash:
            out.append(p)
    return out


def rebuild_one(conn, path: Path, mode: str, size: int) -> int:
    doc_id, doc_hash = path.stem, sha(path.read_text(encoding="utf-8", errors="replace"))
    chunks = load_chunks(path, mode, size)
    if not chunks:
        print(f"!! {path.name} 切出 0 条：多半是扫描件或解析退化，跳过并保持旧版本在线")
        return 0
    vecs = [normalize(v) for v in embed([c.text for c in chunks])]
    t0 = time.perf_counter()
    with conn.transaction():                                   # 一个事务：写入 + 切换，原子
        for c, v in zip(chunks, vecs):
            conn.execute("""
                INSERT INTO chunks (chunk_id, doc_id, seq, heading, content, tokens, metadata, embedding,
                                    doc_hash, active, indexed_at)
                VALUES (%(cid)s, %(doc)s, %(seq)s, %(head)s, %(txt)s, %(tok)s, %(meta)s::jsonb, %(emb)s::vector,
                        %(dh)s, true, now())
                ON CONFLICT (chunk_id) DO UPDATE
                SET active = true, doc_hash = EXCLUDED.doc_hash, indexed_at = now(),
                    embedding = EXCLUDED.embedding, content = EXCLUDED.content;
            """, {"cid": c.chunk_id, "doc": doc_id, "seq": c.seq, "head": c.heading, "txt": c.text,
                  "tok": c.tokens, "meta": '{"sec_level": 1}', "emb": "[" + ",".join(f"{x:.6f}" for x in v) + "]",
                  "dh": doc_hash})
        stale = conn.execute(
            "DELETE FROM chunks WHERE doc_id=%(d)s AND doc_hash <> %(dh)s AND active",
            {"d": doc_id, "dh": doc_hash}).rowcount             # 旧内容直接删（同 doc 的过期版本留着只会继续命中）
        print(f"{doc_id}: 新 {len(chunks)} 条，清理旧版 {stale} 条，{time.perf_counter() - t0:.1f}s")
    return len(chunks)


def main() -> None:
    ap = argparse.ArgumentParser()
    ap.add_argument("--data", default="data/raw")
    ap.add_argument("--since", type=int, help="只处理最近 N 天修改过的候选文件（mtime 预筛）")
    ap.add_argument("--all", action="store_true")
    ap.add_argument("--dry-run", action="store_true")
    ap.add_argument("--mode", default="structured")
    ap.add_argument("--size", type=int, default=400)
    a = ap.parse_args()

    with psycopg.connect(os.environ["PG_DSN"]) as conn:
        docs = candidates(Path(a.data), None if a.all or a.since is None else a.since)
        todo = changed(conn, docs)
        est = sum(len(c.tokens) for p in todo for c in load_chunks(p, a.mode, a.size))
        print(f"候选 {len(docs)} 篇，内容实际变更 {len(todo)} 篇，预计嵌入 token {est}")
        if a.dry_run:
            for p in todo:
                print("  待重建:", p.name)
            return
        for p in todo:
            rebuild_one(conn, p, a.mode, a.size)
    print("完成。别忘了：python metrics.py  # 索引变更后指标必须复测，阈值也要看 gate 的 index_fingerprint")


if __name__ == "__main__":
    main()
```

表要加的三列：`doc_hash text`、`active boolean NOT NULL DEFAULT true`、`indexed_at timestamptz`；检索 SQL 全部补 `AND active`（W13/W14 所有查询语句都要改，改漏一处就会命中过期版本——今天用 `grep -rn "FROM chunks" src/` 自查）。

**今天的动作**

- [ ] 给 `chunks` 加三列，回填 `doc_hash`，并把全部 `active=true`。
- [ ] 造一次真实变更：改某文档的一段、加一段、删一段，跑 `reindex.py --since 1`，确认只重建 1 篇。
- [ ] 用 `touch`（只改 mtime 不改内容）验证第二层哈希判据生效：应显示"内容实际变更 0 篇"。
- [ ] 重跑后 `SELECT count(*) WHERE doc_id=X AND active` 只有一份新版本；旧 chunk 不残留。
- [ ] 全部检索 SQL 补 `AND active`，grep 自查无遗漏，`metrics.py` 复跑并记录指标是否变化。

**会踩的坑**

1. **只按 mtime 判变更**：同步盘一天改一万次时间戳，你为没变的文档付了整批 embedding 费。
2. **新旧版本并存于检索路径**：改完没软删旧行，用户问到的还是上周那版条款，而且两版都进 top-5 互相矛盾。诊断：`SELECT doc_id, count(DISTINCT doc_hash) FROM chunks WHERE active GROUP BY 1 HAVING count(*)>1`。
3. **切换不在事务里**：新 chunk 嵌入调用中途失败，旧的已置为 inactive → 该文档彻底查不到。就是那条 `with conn.transaction()` 的作用。
4. **索引变更后没重跑评测**：一周后指标下滑，你去找模型问题，其实是文档变更导致某条标注的 `relevant_ids` 已不存在。

**自测**

1. `--since` 为什么要两层判据（mtime + 内容哈希）？各挡住什么？
2. 软删与硬删的边界在哪：什么数据永远该留、什么必须真删？
3. 为什么 `reindex` 之后拒答阈值也要复看？（提示：`index_fingerprint`。）

---

## 周六 · 深度块（120 分钟）：全部接入项目一，交付 v2

目标：项目一从"日志排障助手"升级为"带证据、会拒答、有记忆的排障助手"。改动只加不改：**入口签名 `answer(question, session_id)` 不动**（W11 那天钉死的变更面今天兑现），工具注册表加 `rag_search`，系统提示装配处插 `load_block()`，出口加引用与闸门。

### 第一段 55' · 接线

```python
"""projects/p1/graph.py 的关键改动骨架（只列增量，其余不动）"""
from __future__ import annotations

from langgraph.checkpoint.sqlite import SqliteSaver      # pip install langgraph-checkpoint-sqlite

from citations import Evidence, parse_answer
from gate import Threshold
from memory.store import load_block
from reindex_guard import assert_index_fresh             # 你周五那个指纹检查，起个 5 行的实现
from tools.rag_search import RagSearch, TOOL_SPEC

TH = Threshold.model_validate_json(open("config/threshold.json", encoding="utf-8").read())


def build(dsn: str, saver):
    rag = RagSearch(dsn, TH)
    registry = {"search_logs": search_logs, "read_log_context": read_log_context,
                "get_service_meta": get_service_meta, "rag_search": rag}      # <- 唯一新增工具
    builder = build_graph(registry)                      # W10/W11 那套不动
    return builder.compile(checkpointer=saver)


def answer(question: str, session_id: str, user_id: str = "default") -> dict:
    """入口签名保持稳定，user_id 用默认值兼容 W12 已有调用。"""
    cfg = {"configurable": {"thread_id": f"user:{user_id}:chat:{session_id}"}, "recursion_limit": 16}
    facts = load_block(user_id)                          # L2 记忆：每轮硬注入，不靠摘要
    assert_index_fresh(TH.index_fingerprint)             # 索引换过版本就报错，不许用过时的拒答阈值
    state = app.invoke(
        {"question": question, "messages": [{"role": "user", "content": question}],   # 只传增量（W12 军规）
         "observations": [], "repairs": 0, "system_prefix": facts},
        cfg)
    ans = parse_answer(state["final_text"], state.get("evidences", []))
    return {"answer": ans.text, "citations": ans.citations, "fabricated": ans.fabricated,
            "refused": state.get("gate_status") == "refuse", "session_id": session_id}
```

三处最容易接线错误的地方，先在这里说破：

- **`refused` 要在生成之前判**：闸门在检索工具内部给出 `status`，节点路由据此决定是否调用生成。如果在生成之后才"看看答案像不像拒答"，等于什么都没做。
- **`evidences` 必须是本轮的**：从 state 里取，不能从进程级缓存取（W14 引用坑 #3 的编号错位）。
- **`thread_id` 里带 `user_id`**：会话隔离与记忆隔离用同一个主键，少一处判据。

### 休息 10'（离开屏幕）

### 第二段 55' · 端到端演示 + README

演示脚本（照着走一遍，每步截图或终端录屏存进 `demo/`）：

1. `python reindex.py --since 1` → 显示变更 0 篇（证明幂等与哈希判据）。
2. 问一条**库里有**的题（标注集里选）→ 输出带 `[1][2]`，`render()` 展开原文，人工核一条。
3. 问一条**负样本**（"导出上限 3 万行怎么特批"）→ 明确拒答"资料中未说明"，且没有编造条款。**这一步是本周的验收点，也是演示里最出彩的一步。**
4. 换一个 `user_id` 问同一条 → 事实注入不同，证明记忆隔离。
5. `Ctrl+C` 杀掉进程，重启，同 `session_id` 续问"刚才那条的第二步是什么" → checkpointer 恢复成功。
6. 故意越权：用 `bob` 问 alice 的资料 → 检索层 0 条（W14 周五的红队断言跑在项目一里）。

README 新增/改写小节（写给"五分钟要看懂"的读者）：架构图上加一层"检索子系统（混合 + 阈值 + ACL）"；把 W14 三方案对比表**原样搬进来**并给一句话结论；新增"如何核查一条回答"（引用展开方式）与"拒答策略与阈值标定方法"；安全小节写清"权限在检索层、缺字段 fail-closed、trace 落盘脱敏（未完成，见 backlog）"；已知问题里必须有一条真实缺陷（比如"inferred 记忆未实现人工复核队列"）。

- [ ] 六步演示全部走通，第 3、6 步任一失败就回头修，不许"演示时手快绕过"。
- [ ] 用 `experiments/w14_compare.md` 的最优路径作为线上默认，把配置项名字写进 README。
- [ ] README 至少新增三节，并把"引用如何核查"配一张终端截图。
- [ ] 跑一次完整 20 条标注集端到端（不只是检索层），记 `answer-level` 通过率——W21 的评测集就是这个的放大版。
- [ ] `learning-log.md` 三行 + 本周状态（完成 / 部分（缺什么）/ 未做），提交，打 tag `p1-v2`。

**会踩的坑**

1. **检索工具与原有工具抢调用**：模型遇事先查文档库（有资料可查的诱惑大于跑日志工具）。修法：`rag_search` 的 description 收窄（"仅当需要制度/参数/流程定义时"），并在系统提示里给顺序偏好；观察 trace 里的 tool_call 分布确认。
2. **拒答在项目一里从不触发**：阈值文件读到了默认值而不是你标定的值（`Threshold.model_validate` 字段名对不上被静默忽略）。诊断：启动时打印一次生效阈值。
3. **checkpointer 与手动拼历史并存**：消息翻倍、模型复读（W12 军规第一条）。
4. **演示用新索引、评测用旧标注**：`relevant_ids` 对不上，指标假崩。演示前跑一次 `metrics.py` 建立一致性证明。

---

## 本周术语卡

| 术语 | 一句话解释 | 为什么重要 |
|---|---|---|
| 引用标记 citation | 答案里指向本轮证据集的编号 | 让"有没有编造"变成可自动判定的事 |
| 证据集 evidence set | 本轮实际送进 prompt 的片段与其短标签映射 | 引用校验的白名单，必须与轮次绑定 |
| 拒答 refusal | 分数不过阈值时不生成，输出"资料未说明" | 补上 ANN 永远返回 top-k 的结构缺陷 |
| 置信阈值 confidence gate | 用正/负例分数分布定标出的放行线 | 未标定的阈值等于随机数；且随索引版本漂移 |
| 负样本 negative query | 库里确实没有答案的问题 | 没有它你无法证明模型不瞎编 |
| 万能段 boilerplate chunk | 跟什么都像的总则性片段 | 长期霸占 top-1、把闸门顶开的主要来源 |
| Agentic RAG | 检索作为工具，模型自主决定时机与查询 | 多跳与自纠有效；简单题更慢更贵，需次数熔断 |
| 短期记忆 / checkpointer | 会话状态在节点边界快照，按 thread_id 恢复 | 跨进程续聊，但它不是长期记忆 |
| 长期事实记忆 | 结构化 `user_memory` 表，按 key upsert | 关键约束的唯一可靠载体，不靠摘要 |
| 语义记忆 | 历史交互沉淀成的可检索向量 | 召回方式同 RAG，必须按 user_id 过滤 |
| 来源与置信 source/confidence | 事实是用户声明、工具验证还是模型推断 | 决定能否覆盖旧值，防臆测污染 |
| 软删除与切换 soft delete | 旧版本置 `active=false`，新版本原子切换 | 蓝绿发布搬到索引层，失败可回滚 |
| 内容指纹 doc_hash | 文件内容的哈希，作为变更判据 | mtime 会变而内容不变；只有哈希能省掉重嵌入 |

## 延伸阅读（可选，不占 1 小时额度）

LangGraph 官方文档的 Persistence / Checkpointers 一节（只看 `get_state`、`thread_id` 语义与官方对 checkpointer 适用边界那两段）；你服务商 rerank 与 embeddings 的限流页（本周脚本大量调用，先看清 RPM）。其余一律不看。
