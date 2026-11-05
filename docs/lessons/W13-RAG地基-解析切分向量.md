# W13 · RAG 的地基：数据、切分、向量

> 本周在计划中的位置：阶段 3 第一周，把"知识从模型参数搬到外部库、推理时实时检索注入"这条链路从零搭通，周六交付 `build_index.py` + `search.py`。

## 本周导航

| 日 | 主题 | 今天的产出物 |
|---|---|---|
| 周一 | RAG 心智模型 + 什么时候**不该**用 | 一页《RAG / 长上下文 / 微调 取舍判据》 |
| 周二 | 真实文档解析与清洗 | 9 份 clean 文档 + 一份脏点清单 |
| 周三 | 切分三档实验（固定 / 递归 / 结构） | `chunker.py` + 每档 3 组参数的统计与样例 |
| 周四 | Embedding：维度、归一化、相似度换算 | `emb_probe.py` + 同义/无关距离验证表 |
| 周五 | 向量存储落地（pgvector 为主） | 建表 + HNSW 索引 + chunk/向量/元数据入库 |
| 周六 | 串成一条链 | `build_index.py`（可增量重跑）+ `search.py`（top-k CLI） |
| 周日 | 休息，不许补课 | — |

本周不追求检索质量，只要求每一环你都亲手摸过。质量是 W14 的主题。

---

## 周一 · RAG 是什么，以及它什么时候是错的答案

**本质**

RAG（检索增强生成，Retrieval-Augmented Generation）只做一件事：把"知识存在哪里"从模型参数迁到一张你能增删改查的外部库，每次推理时按当前问题捞回最相关的几段塞进上下文窗口。模型没有变聪明，它只是开卷考试了。

所以 RAG 的收益上限由两件事决定：检索能不能捞到那一段（W14 的全部主题），以及模型能否从捞到的片段读出答案。前者是工程问题，后者是模型能力问题——分清楚能省掉大量无用调参。

**Java 视角**

模型参数是一个不可变的静态配置，向量库是一张只读从库表。RAG 本质上就是在 Service 层加一次 `SELECT ... ORDER BY distance LIMIT k`，再把结果集渲染进请求体。你写的不是算法，是一个带模糊查询的读接口 + 一个装配器。

**核心讲解**

一次 RAG 全生命周期分两段。**索引期**（离线）：解析 → 清洗 → 切 chunk → 每个 chunk 过 embedding → 向量 + 原文 + 元数据入库。**查询期**（在线）：问题过同一个 embedding → 向量库做 ANN（近似最近邻）搜索取 top-k → 排序/阈值过滤 → k 段原文拼进 prompt → 生成。

三个必须记死的机制细节：

1. **问题与文档必须用同一个 embedding 模型、同一维度**。换模型 = 旧向量全部作废 = 全量重建索引。这是 RAG 最常见的"改一处炸全站"。
2. **chunk 是检索的原子单位**。你检索不了半句话，也检索不了一整本产品手册。切分决定了答案粒度，之后所有优化都在这个粒度上打转。
3. **窗口是硬预算**。k 段 × 每段 token + 历史 + 系统提示，超了就截断。chunk 切大了，top-5 就把窗口吃光，还得给输出留位置。

什么时候**不该**用 RAG——这部分比"该用"更有面试价值：

| 场景 | 为什么 RAG 是错的 | 该怎么做 |
|---|---|---|
| 纯推理任务（给报错栈找根因、JSON 转 SQL） | 知识已在输入里，缺的是推理，检索只塞噪声 | 直接调模型，把时间花在上下文工程 |
| 语料小且稳定（3000 字规范、错误码表） | 检索多余，切分反而破坏完整性 | 整份进 system prompt + prompt caching |
| 全局聚合（"统计上季度所有工单高频问题"） | top-k 看见的是样本不是分布 | SQL / 批处理，结果再交模型汇总 |
| 要能力或风格而非知识（固定话术、团队规范） | 参数里学不到的，检索补不进 | 提示词先行，不够再谈微调 |
| 答案分散且必须齐全（"列出全部退款条款"） | recall 被 chunk 粒度卡死，永远漏 | map-reduce 遍历全量或结构化查询 |

RAG vs 长上下文 vs 微调的四条判据：**知识体量**（能整份塞进窗口就先塞，粗略按 1 个汉字 ≈ 1 token 估，各 tokenizer 有出入）；**变更频率**（日更的东西只能进外部库，微调永远追不上）；**单次成本**（长上下文每次为全部 token 付费，检索只为 top-k 付费，高 QPS 差一个数量级；但 prompt caching 能把"固定语料整份塞"的成本大幅拉低，语料不大且可缓存时长上下文更优）；**瓶颈定位**（答错时先看该依据的那段有没有进 top-k：进了还没答对是生成问题，没进才是检索问题）。

微调的诚实定位：微调改**行为**（格式、语气、工具调用习惯），RAG 改**知识**。用微调补知识是当前最常见的误用——慢、贵、且无法追溯来源。

**代码**

```python
"""rag_or_not.py — 把"该不该上 RAG"写成可回归检查的判断，而不是一页会腐烂的 PPT。"""
from __future__ import annotations

from enum import Enum
from pydantic import BaseModel, Field


class Route(str, Enum):
    PLAIN_LLM = "直接调模型"
    STUFF_PROMPT = "整份塞提示词(+缓存)"
    RAG = "检索增强"
    STRUCTURED = "结构化查询后汇总"


class Scenario(BaseModel):
    corpus_tokens: int
    daily_change_ratio: float = Field(ge=0, le=1)
    qps: int = Field(ge=0)
    reasoning_only: bool = False
    need_full_coverage: bool = False       # "全部条款"类需求
    context_window: int = 128_000


def decide(s: Scenario) -> tuple[Route, str]:
    if s.reasoning_only:
        return Route.PLAIN_LLM, "知识已在输入中，检索只会稀释上下文"
    if s.need_full_coverage:
        return Route.STRUCTURED, "top-k 结构上无法保证覆盖齐全"
    # 语料小且稳定：留 70% 窗口给历史与工具结果，0.3 是经验起点，W14 有数据再调
    if s.corpus_tokens <= s.context_window * 0.3 and s.daily_change_ratio < 0.05:
        return Route.STUFF_PROMPT, "整份塞入 + 缓存优于检索"
    if s.daily_change_ratio >= 0.3 or s.corpus_tokens > s.context_window * 0.3:
        return Route.RAG, "知识量大或高频变动，只有外部库跟得上"
    return Route.RAG, "默认走检索"


if __name__ == "__main__":
    cases = {
        "3000 字错误码表，季度更新，低 QPS": Scenario(corpus_tokens=5_000, daily_change_ratio=0.0, qps=1),
        "60 万字运维手册，日更": Scenario(corpus_tokens=800_000, daily_change_ratio=0.02, qps=5),
        "给一段报错栈找根因": Scenario(corpus_tokens=2_000, daily_change_ratio=0.0, qps=5, reasoning_only=True),
    }
    for name, sc in cases.items():
        route, why = decide(sc)
        print(f"{name} -> {route.value} | {why}")
```

**今天的动作**

- [ ] 用自己的话（不许复制）写 3 行：RAG 搬走了什么、留下了什么。
- [ ] 跑通 `rag_or_not.py`，再加 2 个**你工作里的真实场景**；判断与预期不符就是权重该改。
- [ ] 抄一份"什么时候不该用 RAG"清单进项目 README，作为自己项目的评审门槛。
- [ ] 选定本周语料：PDF / HTML / Markdown 各 3 份，全部真实文档，放 `agent-lab/data/raw/`。
- [ ] 确认这 9 份覆盖项目一那个 Agent 的真实提问范围，否则 W14 的标注集没有意义。

**会踩的坑**

1. **拿 RAG 修"模型不听话"**：格式乱、不调工具，你却去调 chunk size。诊断：把该依据的那段直接贴进 prompt 再问一次，问题依旧就说明与检索无关。
2. **切完 chunk 人工没看过一眼**。所有"检索不准"的第一排查动作是打开 chunk 文件肉眼看那段在不在、有没有被切成两半。跳过这步直接换模型是烧钱。
3. **语料选得太干净**。用规整百科练切分会让你以为切分很简单；真实内部文档的脏度是练习材料里没有的。
4. **本周就纠结选哪个 embedding 模型**。任何模型都能跑通链路，先用最便宜的，W14 有评测集你才有资格比较。

**自测**

1. 用户问"我们全部的数据导出审批环节有哪些"，为什么不该用 RAG 回答？改成什么？
2. 同一份语料，top-k 从 5 调到 20，成本涨 4 倍但答案没变好，最可能的原因是什么？怎么验证？
3. 四条判据里哪一条能让你立刻否掉 RAG 方案？为什么？

---

## 周二 · 文档解析与清洗：脏数据是 RAG 失败第一原因

**本质**

解析（Parsing）把 PDF/HTML/Markdown 变成文本，清洗（Cleaning）把"给人看的排版"变成"给检索用的文本"。两者目标冲突：页眉、页脚、水印、目录点线对人是导航线索，对 embedding 是纯噪声，而且是**系统性**噪声——每页都加同样的字，会让所有 chunk 的向量彼此靠近，把区分度直接抹平。

**Java 视角**

等同于把 Oracle 导出的定长报表文本入库前先做 ETL。你不会把带页眉的 CSV 直接插表；这里没有 schema 能约束你，全靠规则。

**核心讲解**

三类来源的脏点及其**检索侧失效表现**：

- **PDF**：页眉页脚（公司名 + 页码）每页重复；分栏被按行合并成左右串行的乱句；跨页断句（上页末"本制度适用于"、下页首"全体员工"）；表格被拉平成一行行碎片，表头在每个数据行附近重复出现，语义尽失；扫描件 PDF 文本层为空，抽出空字符串却"执行成功"。
- **HTML**：导航栏、页脚、Cookie 提示、评论全混进正文；`<script>`/`<style>` 内容被当文本；同一正文在列表页与详情页重复，导致 top-5 全是同一段的不同副本，多样性实际为 1。
- **Markdown**：代码块围栏（```）被从中间劈开，半个函数体独立成 chunk；`|` 表格被切分器当分隔符误切；目录锚点、脚注混入正文。

清洗次序不能反，因为它决定 id 稳定性：① 先按来源抽正文（HTML 走标签白名单，Markdown 保留结构标记，PDF 逐页抽）；② 再去跨页重复噪声——把每行归一化（去空白、数字抹成 `#`）后统计**每页只计一次**的出现次数，出现在 ≥30% 页面且长度 ≤40 字符的行判为模板行删除，这个"重复即模板"启发式对企业 PDF 大多有效；③ 再处理特殊结构——表格转 `表头=值` 的一行一记录（比保留竖线好检索得多），代码块整体保护；④ 最后做文本规范化（NFKC、空白折叠、去 BOM、修乱码）。

乱码要单独说：UTF-8 字节被当 GBK 解码写出的是 `ä¸­æ–‡`，反向则是 `` / ``。诊断方法：清洗前后各存一份，diff 出被丢弃字符，同一位置反复出现 `ï»¿`、`å`、`` 就是解码链错配。修法是按优先级多次试解码，别猜——猜错的文本喂进 embedding，检索会莫名其妙地差。

**代码**

```python
"""cleaning.py — 抽正文之前先按'重复即模板'去噪；顺序不可反。"""
from __future__ import annotations

import re
import unicodedata
from collections import Counter
from pathlib import Path


def fingerprint(line: str) -> str:
    # 抹掉数字：否则每页页码不同，页眉永远统计不到重复
    return re.sub(r"\d", "#", re.sub(r"\s+", "", line))


def drop_boilerplate(pages: list[str], ratio: float = 0.3) -> list[str]:
    counter: Counter[str] = Counter()
    for page in pages:
        counter.update({fingerprint(l) for l in page.splitlines() if l.strip()})
        # 每页只计一次：防止正文里同一句出现 5 次被误判成页眉
    boiler = {k for k, v in counter.items() if v >= max(2, int(len(pages) * ratio)) and len(k) <= 40}
    return ["\n".join(l for l in p.splitlines() if fingerprint(l) not in boiler) for p in pages]


def table_to_kv(md_table: str) -> str:
    """表格拉平成 '表头=值' 记录：embedding 与 BM25 都能命中完整语义行。"""
    rows = [r for r in md_table.strip().splitlines() if r.strip()]
    rows = [r for r in rows if not re.fullmatch(r"\|[\s:\-|]+\|", r)]
    if len(rows) < 2:
        return md_table
    header = [c.strip() for c in rows[0].strip().strip("|").split("|")]
    return "\n".join(
        "; ".join(f"{h}={c}" for h, c in zip(header, [x.strip() for x in r.strip().strip("|").split("|")]) if c)
        for r in rows[1:]
    )


def sanitize(text: str) -> str:
    text = unicodedata.normalize("NFKC", text)   # 全角/兼容字符统一，否则同词不同码点
    text = text.replace("\u3000", " ").replace("\xa0", " ")
    return re.sub(r"\n{3,}", "\n\n", re.sub(r"[ \t]+", " ", text)).strip()


def decode_bytes(raw: bytes) -> str:
    for enc in ("utf-8", "gb18030", "big5"):
        try:
            return raw.decode(enc)
        except UnicodeDecodeError:
            continue
    return raw.decode("utf-8", errors="replace")  # 兜底：替换符会显式暴露问题，别静默吞


def clean_file(path: Path) -> Path:
    pages = decode_bytes(path.read_bytes()).split("\f")   # \f 通常是页边界
    body = "\n\n".join(sanitize(p) for p in drop_boilerplate(pages) if p.strip())
    assert len(body) > 100, f"{path.name}: 抽取结果近乎为空，多半是扫描件，走 OCR 不是今晚的事"
    dest = Path("data/clean") / (path.stem + ".md")
    dest.parent.mkdir(parents=True, exist_ok=True)
    dest.write_text(body, encoding="utf-8")
    print(f"{path.name}: {len(path.read_bytes())}B -> {len(body)} chars")
    return dest
```

PDF 抽取不唯一：`pypdf` 轻、`pdfplumber` 表格更友好。今天的 3 份 PDF 各用两个工具抽一遍并 diff，差异处就是你该写规则的地方。

**今天的动作**

- [ ] 9 份文档全部抽取，原始字符数记进表。
- [ ] 3 份 PDF 用 `pypdf` 与 `pdfplumber` 各抽一次，diff 并记下哪个更适合你的文档形态。
- [ ] 在 clean 结果里**人工找 3 个仍然存在的脏点**（今天最值钱的 10 分钟），写进脏点清单。
- [ ] 表格转 KV 前后各留一个样例，肉眼判断语义是否保住。
- [ ] 提交 `data/clean/` 与脏点清单；不要提交原始 PDF（体积 + 可能有机密）。

**会踩的坑**

1. **页眉删干净了正文也少一段**：阈值 0.3 太激进，某句只出现 2 次却正好是关键条款。诊断：清洗前后对比被删行，抽 20 行人工判该不该删。
2. **扫描件抽出空串而脚本"成功"**：某文档 chunk 数为 0 却不报错。诊断：像上面那样直接 assert，宁可红不要静默。
3. **HTML 正文重复**（列表页摘要 + 详情页全文）：top-5 是同一段的不同副本。诊断：清洗后按内容哈希去重，别靠 URL 猜。
4. **代码块被当普通文本**：函数签名与函数体分家，问"X 怎么调用"命中注释却丢了示例。

**自测**

1. 页眉检测为什么必须"每页只计一次"？改用总出现次数会出什么问题？
2. 表格拉平成 KV 与保留原始 `|` 文本，哪种对 embedding 更友好、哪种对 BM25 更友好？理由分别是什么？
3. 清洗后字符数下降 45%，你现在先检查什么？

---

## 周三 · 切分策略三档与它们的失效场景

**本质**

切分（Chunking）是在检索精度与语义完整之间做取舍：chunk 小 → 向量表征聚焦、命中准，但送进 prompt 的信息可能不完整；chunk 大 → 上下文完整，但要一个向量代表多个主题，相似度被平均掉，还挤占窗口预算。没有最优 size，只有对你的文档结构最不难看的 size。

**Java 视角**

类似决定表该按什么粒度分行：太细则一次查询要捞很多行，太粗则索引选择性差。overlap 是为防边界丢失故意做的冗余存储，代价是总行数与 embedding 费用上升。

**核心讲解**

三档策略与失效场景：

1. **固定长度（fixed-size）**：每 N 个 token 一刀，overlap M。5 行实现、成本可预测。失效：完全无视语义边界，句子被腰斩、条件句的条件与结论分家、列表被拆散。它是 baseline，不是终态。
2. **递归分隔符（recursive splitting）**：按优先级 `["\n\n", "\n", "。", "！", "？", "；", "，", " ", ""]` 从段落降到句子再降到字符，直到落进 size。失效：对结构良好的 Markdown 很好，对没有明显分隔符的文本（OCR 后一整块、纯表格）会退化到接近固定长度；分隔符语言选错等于没分隔（英文语料用中文句号切）。
3. **按结构（structure-aware）**：沿标题层级切，小节标题作为前缀拼进每个 chunk，代码块/表格/列表当原子块不可分。失效：标题层级不规范时（内部 wiki 极常见，全 H1 或用加粗冒充标题）结构树退化成一条链，等于没切；跨小节引用（"见 3.2 节"）被切断。

**把标题拼进 chunk 前缀**性价比极高：`【3.2 导出限额】单次导出上限 5 万行…`。这一行让 embedding 和 BM25 同时拿到章节上下文，代价是每 chunk 多几十字符。

size 与 overlap 的影响**方向**（先验，数值你自己明天之前跑出来）：

- size 100 → 500 token：recall@5 通常先升后降——升是完整语义进来了，降是大 chunk 开始混主题。
- size 500 → 2000 token：recall@5 一般下降（答案被噪声稀释），但答案质量可能上升（上下文更全）。这两个指标经常反向，W14 你会正面撞上。
- overlap 0 → 15%：跨边界丢失明显减少；再加到 40% 收益基本归零，但 chunk 数量与 embedding 成本线性上升，还引入"同一段近似副本挤满 top-k"。
- 中文经验起点：`chunk_size=300~500 token，overlap=10%~15%`，先递归切，再试结构切。

**代码**

```python
"""chunker.py — 三档切分同接口，方便后面做参数对比。"""
from __future__ import annotations

import hashlib
import json
import re
import unicodedata
from pathlib import Path
from pydantic import BaseModel

SEPARATORS = ["\n\n", "\n", "。", "！", "？", "；", "，", " ", ""]


class Chunk(BaseModel):
    chunk_id: str      # 内容哈希：增量重建与引用回传都依赖它稳定
    doc_id: str
    seq: int
    heading: str = ""
    text: str
    tokens: int


def est_tokens(text: str) -> int:
    """粗估：CJK 每字 1 token，其余 4 字符 1 token。够用来比参数。"""
    cjk = sum(1 for ch in text if "\u4e00" <= ch <= "\u9fff")
    return int(cjk + (len(text) - cjk) / 4)


def chunk_hash(text: str) -> str:
    norm = unicodedata.normalize("NFKC", re.sub(r"\s+", "", text))
    return hashlib.sha1(norm.encode()).hexdigest()[:16]


def split_fixed(text: str, size: int = 400, overlap: int = 60) -> list[str]:
    step = max(1, size - overlap)
    return [text[i:i + size] for i in range(0, len(text), step)]


def split_recursive(text: str, size: int = 400, overlap: int = 60, seps: list[str] | None = None) -> list[str]:
    seps = seps or SEPARATORS
    if est_tokens(text) <= size or not seps:
        return [text] if text.strip() else []
    sep, rest = seps[0], seps[1:]
    # 用前视断言保留分隔符本身：中文句号丢了会破坏句子完整性
    parts = [p for p in re.split(f"(?<={re.escape(sep)})" if sep else r"(?=.)", text) if p]
    if len(parts) == 1:
        return split_recursive(text, size, overlap, rest)
    merged: list[str] = []
    buf = ""
    for p in parts:
        if est_tokens(buf + p) <= size:
            buf += p
        else:
            if buf:
                merged.append(buf)
            buf = (buf[-overlap:] + p) if overlap else p   # overlap 保住跨边界语义
    if buf:
        merged.append(buf)
    return [x for m in merged for x in (split_recursive(m, size, overlap, rest) if est_tokens(m) > size else [m])]


def split_structured(md: str, size: int = 400, overlap: int = 60) -> list[tuple[str, str]]:
    blocks: list[tuple[str, list[str]]] = []
    heading, buf, in_fence = "", [], False
    for ln in md.splitlines():
        in_fence = in_fence if False else (not in_fence if ln.startswith("```") else in_fence)
        if not in_fence and re.match(r"^#{1,4}\s", ln):
            if buf:
                blocks.append((heading, buf))
            heading, buf = ln.strip("# "), []
        else:
            buf.append(ln)
    if buf:
        blocks.append((heading, buf))
    out: list[tuple[str, str]] = []
    for h, buf in blocks:
        body = "\n".join(buf).strip()
        prefix = f"【{h}】" if h else ""
        if not body:
            continue
        if est_tokens(prefix + body) <= size:
            out.append((h, prefix + body))
        else:   # 大节内部再递归切，但每块都带同一标题前缀，章节信息不丢
            out.extend((h, prefix + piece) for piece in split_recursive(body, size, overlap))
    return out


def build_chunks(text: str, doc_id: str, mode: str = "recursive", size: int = 400, overlap: int = 60) -> list[Chunk]:
    pairs = (
        [("", t) for t in split_fixed(text, size, overlap)] if mode == "fixed"
        else split_structured(text, size, overlap) if mode == "structured"
        else [("", t) for t in split_recursive(text, size, overlap)]
    )
    return [Chunk(chunk_id=chunk_hash(t), doc_id=doc_id, seq=i, heading=h, text=t, tokens=est_tokens(t))
            for i, (h, t) in enumerate(pairs) if t.strip()]


if __name__ == "__main__":
    rows = []
    for f in sorted(Path("data/clean").glob("*.md")):
        text = f.read_text(encoding="utf-8")
        for mode in ("fixed", "recursive", "structured"):
            for size in (200, 400, 800):
                cs = build_chunks(text, f.stem, mode, size, size // 8)
                lens = [c.tokens for c in cs] or [0]
                rows.append({"doc": f.stem, "mode": mode, "size": size, "n": len(cs),
                             "avg": round(sum(lens) / len(lens), 1), "min": min(lens),
                             # tiny_ratio：碎片 chunk 占比，越高说明检索越容易被无意义片段占位
                             "tiny_ratio": round(sum(x < 30 for x in lens) / len(lens), 3)})
    Path("experiments/w13_chunk_stats.jsonl").write_text(
        "\n".join(json.dumps(r, ensure_ascii=False) for r in rows) + "\n", encoding="utf-8")
    for r in rows[:6]:
        print(r)
```

**今天的动作**

- [ ] 三模式 × 三档 size（200/400/800）跑完，输出 `experiments/w13_chunk_stats.jsonl`。
- [ ] 每档挑 2 个 chunk 肉眼查：句子腰斩、代码块劈开、表格半截各有几个。
- [ ] 把最难看的一个 chunk 粘进笔记并标注为什么难看——它就是明天入库后的检索失败候选。
- [ ] 给 chunk 加标题前缀，对比加前缀前后的 `avg`，确认成本可接受。
- [ ] 定本周默认参数并写理由（如 `structured / 400 / 50`），W14 有指标后再改。

**会踩的坑**

1. **overlap 太大造成近似副本**：top-5 是同一段的错位拷贝，实际覆盖量等于 1 条。诊断：`len(set(ids)) == k` 但内容高度相似；降到 10% 复测。
2. **`min` 出现个位数**：分隔符表尾部（空格、空串）造出碎片 chunk，向量无意义却能进 top-k。诊断：看 `tiny_ratio`，对 <30 token 的块做合并或丢弃。
3. **中文抄了英文默认分隔符表**：切出整块或乱切，`n` 异常小。诊断：句子边界是否随机。
4. **代码块未受保护**：chunk 里出现未闭合的 ```。诊断：断言每个 chunk 的 fence 计数为偶数。

**自测**

1. 结构切分时把标题拼进 chunk 前缀，同时改善了检索链路的哪两个环节？
2. 为什么 `chunk_size` 变大可能让 recall@5 下降却让答案质量上升？
3. 同一份 OCR 文本上递归切分与固定切分结果几乎一样，最可能的原因？

---

## 周四 · Embedding：维度、归一化、相似度到距离的换算

**本质**

Embedding 模型把一个 chunk 映射成固定维度（dimension，常见 384/768/1024/1536/3072）的实数向量，语义相近的文本方向相近。它编码的不是"意思"，而是"这个模型认为哪些文本会出现在相似语境"——所以术语密集的内部文档，通用 embedding 常常表现平平。

**Java 视角**

当成一个不可逆的**有损相似度哈希**：相似输入产生相近输出。于是检索退化成 `ORDER BY distance(vec, ?) LIMIT k`，是空间索引问题，跟用 MySQL 做"附近门店"查询同一类事。

**核心讲解**

四个度量与它们必须匹配的原因：

- **余弦相似度（cosine similarity）**：`sim = a·b / (|a||b|)`，只看方向，∈ [-1, 1]，越大越像，文本检索的事实标准。
- **余弦距离**：`1 - sim`，∈ [0, 2]，越小越近。pgvector 的 `<=>` 返回这个，所以 `sim = 1 - <=>`。
- **L2 距离（欧氏）**：`|a-b|`。若向量已 L2 归一化，`|a-b|² = 2 - 2·sim`，即 `d = sqrt(2(1-sim))`，是 sim 的单调递减函数——**归一化后 L2 排序与余弦排序完全一致**。这条换算就是阈值在两种度量之间迁移的依据。
- **内积（inner product）**：`a·b`，未归一化时受模长影响（长文本向量模长偏大会不公平占优）。pgvector 的 `<#>` 返回**负内积**，因为索引只支持升序取 top。

实用结论：模型输出不是单位向量就自己除一下；**建索引的操作符类必须与查询算子一致**，否则 pgvector 不走索引退化成全表扫。

调用侧：`openai` 1.x 是 `client.embeddings.create(input=[...], model=...)`，返回 `data[i].embedding`（`list[float]`）；`text-embedding-3-small` 输出 1536 维、`text-embedding-3-large` 输出 3072 维，可传 `dimensions` 降维（省存储但损质量）。各家字段与批大小上限差异大，**以当前服务商文档为准**。入库按 batch=64 分批 + 失败重试。

今天必做的验证实验：3 组三元组 `原句 / 同义改写 / 关键词重叠但主题无关`，比较两个 sim，合格标准是同义 > 无关（经验区间：同义 0.80~0.95，无关 0.25~0.55，随模型浮动）。再加对抗样例："订单超时时间为 30 分钟" vs "订单超时时间为 60 分钟"——这两句 sim 会**极高**（常 >0.95），因为 embedding 对数值差异不敏感。这个现象必须亲眼看到：它是 RAG 不能单独做数值判断的证据，也是元数据过滤存在的理由。

**代码**

```python
"""emb_probe.py — 不绑 SDK：走 OpenAI 兼容的 /embeddings 端点（httpx）。"""
from __future__ import annotations

import math
import os
from typing import Any

import httpx

BASE = os.environ.get("EMB_BASE_URL", "https://api.openai.com/v1").rstrip("/")
MODEL = os.environ.get("EMB_MODEL", "text-embedding-3-small")
EXPECT_DIM = {"text-embedding-3-small": 1536, "text-embedding-3-large": 3072}.get(MODEL, 1536)


def embed(texts: list[str], batch: int = 64) -> list[list[float]]:
    """分批调用并严格保持与输入同序同长；维度断言挡住'换模型没重建索引'。"""
    out: list[list[float]] = []
    with httpx.Client(timeout=60.0, headers={"Authorization": f"Bearer {os.environ['EMB_API_KEY']}"}) as c:
        for i in range(0, len(texts), batch):
            payload: dict[str, Any] = {"model": MODEL, "input": texts[i:i + batch]}
            data = c.post(f"{BASE}/embeddings", json=payload).raise_for_status().json()["data"]
            # 兼容网关不保证顺序，只有 index 字段是契约，必须按它回填
            out.extend(item["embedding"] for item in sorted(data, key=lambda d: d["index"]))
    assert len(out) == len(texts), "向量条数与输入不符：批量被截断"
    assert all(len(v) == len(out[0]) for v in out), "同一批内维度必须一致"
    return out


def normalize(vec: list[float]) -> list[float]:
    n = math.sqrt(sum(x * x for x in vec))
    return [x / n for x in vec] if n else vec   # 零向量保持零，别抛除零


def cosine(a: list[float], b: list[float]) -> float:
    return sum(x * y for x, y in zip(a, b))     # 归一化后点积即余弦相似度


def l2(a: list[float], b: list[float]) -> float:
    return math.sqrt(sum((x - y) ** 2 for x, y in zip(a, b)))


if __name__ == "__main__":
    triplets = [
        ("用户导出的数据文件 7 天后自动删除", "导出任务生成的文件保留 7 天后会被清理", "用户需登录才能导出数据文件"),
        ("审批人离职后工单自动转给其上级", "若审批人离职，工单流转至直属上级处理", "离职员工需办理工作交接手续"),
    ]
    probes = ["订单超时时间为 30 分钟", "订单超时时间为 60 分钟"]
    vecs = [normalize(v) for v in embed([t for tri in triplets for t in tri] + probes)]
    print(f"model={MODEL} dim={len(vecs[0])}（期望 {EXPECT_DIM}）")
    for i, (a, b, c) in enumerate(triplets):
        va, vb, vc = vecs[i * 3:i * 3 + 3]
        print(f"同义 sim={cosine(va, vb):.4f} l2={l2(va, vb):.4f} | 无关 sim={cosine(va, vc):.4f} l2={l2(va, vc):.4f}")
    v1, v2 = vecs[-2:]
    sim = cosine(v1, v2)
    print(f"对抗样例 30min vs 60min: sim={sim:.4f} <- 数值差异被 embedding 忽略")
    print(f"换算校验 预测 l2={math.sqrt(2 * (1 - sim)):.6f} 实测 l2={l2(v1, v2):.6f}")
```

最后一行是今天的验收：两个数应在 1e-4 内相等。不相等说明向量没真正归一化，后面所有跨度量换算都会偏。

**今天的动作**

- [ ] 跑通 `emb_probe.py`，确认维度与官方声明一致。
- [ ] 记录 3 组三元组 sim 表；有"无关 > 同义"的反例就单独记进笔记（它是 W14 标注集的种子）。
- [ ] 跑对抗样例，亲眼确认 sim 极高，把结论抄进本周笔记。
- [ ] 验证 `l2 == sqrt(2(1-sim))` 一致到 1e-4。
- [ ] 抽 5 个昨天的真实 chunk 生成向量存 `data/emb_probe.jsonl`，明天入库直接用。

**会踩的坑**

1. **换模型只重建了一半索引**：新旧向量混存，尺度不同，top-k 变随机。诊断：`SELECT DISTINCT vector_dims(embedding) FROM chunks;` 一条 SQL 看出有几个维度（函数名以当前版本为准）。
2. **忘了归一化却用内积排序**：长 chunk 系统性排前面。诊断：sim 与长度做散点，正相关就有问题。
3. **把 cosine 阈值直接当 L2 阈值**：尺度不同。诊断：阈值配置项显式命名 `COSINE_DIST_MAX` / `L2_DIST_MAX`，禁止混用。
4. **相信批量返回顺序**：向量与原文错位，命中 A 返回 B，极难查。诊断：按 `index` 回填 + 长度断言。

**自测**

1. 归一化后为什么 L2 与余弦排序结果相同？推出那条等式。
2. 同一段落用两个不同 embedding 模型都入库，为什么检索分数不可比？
3. "北京天气很好" 与 "北京房价很高" 相似度可能意外地高，这暴露了 embedding 什么局限？对 RAG 设计有什么影响？

---

## 周五 · 向量存储落地：pgvector 为主，Qdrant 为备

**本质**

向量库要解决：给定 query 向量，在大量候选里快速找距离最小的 k 个，同时支持"先按元数据过滤再排序"。前一半靠近似索引，后一半正是你熟的那部分——SQL 谓词。

**Java 视角**

pgvector 就是 PostgreSQL 的一个类型 + 索引扩展，用法与 `geo` 类型 + GiST 索引做附近门店查询一模一样：一张表、一个 `vector(1536)` 列、一个 `ORDER BY col <=> ?`，其余全是普通 SQL。备份、事务、权限、迁移全部复用现有认知——这是它在本阶段的最大价值。

**核心讲解**

`CREATE EXTENSION vector;` 之后可用 `vector(dim)`（最多 2000 维）、`halfvec(dim)`（最多 4000 维，半精度省一半空间）、`sparsevec`。字面量就是 `'[0.1,0.2,0.3]'::vector`，维度不符的插入直接报错（好事）。

距离算子：`<->` L2、`<#>` 负内积、`<=>` 余弦距离。索引操作符类必须配套：`vector_l2_ops` / `vector_ip_ops` / `vector_cosine_ops`。**建了 cosine 索引却用 `<->` 查询，索引不会被使用**。

两种索引选择依据：

| | IVFFlat | HNSW |
|---|---|---|
| 建索引速度 | 快 | 慢（官方：更好 recall 换构建时间与写入速度） |
| 内存 | 低 | 高（图常驻） |
| 调参 | `SET ivfflat.probes = 10`（默认 1，越大越准越慢） | `SET hnsw.ef_search = 100`（默认 40，同上） |
| 频繁增删 | 聚类中心漂移，需定期 REINDEX | 增量友好 |
| 前提 | 需已有数据训练聚类 | 空表可建；官方明言"表很小时全表扫可能更快" |

`lists` 起步按官方口径 `rows / 1000`（百万行内），`m` 默认 16、`ef_construction` 默认 64。**本周直接 HNSW + `vector_cosine_ops`**：数据小、反复重跑索引、不折腾聚类。

一条带过滤的相似度查询：

```sql
SELECT chunk_id, doc_id, heading, left(content, 120) AS preview,
       1 - (embedding <=> %(q)s::vector) AS score      -- score 单独算，别用距离当相似度
FROM chunks
WHERE doc_id = ANY(%(docs)s)
  AND coalesce((metadata->>'sec_level')::int, 0) <= %(max_sec)s
ORDER BY embedding <=> %(q)s::vector                    -- 距离升序 = 相似度降序
LIMIT 5;
```

**过滤与 ANN 的相互作用是本周最重要的工程细节**：HNSW 在图遍历中边取边过滤，谓词若滤掉大部分行，为凑齐 LIMIT 会遍历很久，甚至返回不足 k 条。处置：选择率高时调大 `hnsw.ef_search`；数据量 <1 万行干脆不建索引，全表扫更快更准。这是可测量的，明天你自己复现。

Qdrant 备选：`docker run -d -p 6333:6333 -p 6334:6334 -v "$(pwd)/qdrant_storage:/qdrant/storage" qdrant/qdrant`；`QdrantClient(url="http://localhost:6333")`，本地嵌入模式 `QdrantClient(path="data/qdrant")` 官方定位为开发/原型/测试用。它 payload 索引与过滤表达力更强，代价是多运维一个组件、权限过滤要自己测。

**代码**

```python
"""vector_store.py — psycopg 3 + 原生 SQL，不引 ORM。"""
from __future__ import annotations

import json
import os
from dataclasses import dataclass
from typing import Any

import psycopg

DDL = """
CREATE TABLE IF NOT EXISTS chunks (
    chunk_id   text PRIMARY KEY,              -- 内容哈希：重跑天然幂等
    doc_id     text NOT NULL,
    seq        int  NOT NULL,
    heading    text NOT NULL DEFAULT '',
    content    text NOT NULL,
    tokens     int  NOT NULL,
    metadata   jsonb NOT NULL DEFAULT '{}'::jsonb,
    embedding  vector(1536) NOT NULL,         -- 维度写死：换模型必报错，不许静默混存
    updated_at timestamptz NOT NULL DEFAULT now()
);
CREATE INDEX IF NOT EXISTS chunks_doc_idx  ON chunks (doc_id);
CREATE INDEX IF NOT EXISTS chunks_meta_idx ON chunks USING gin (metadata);
"""
HNSW = ("CREATE INDEX IF NOT EXISTS chunks_emb_idx ON chunks "
        "USING hnsw (embedding vector_cosine_ops) WITH (m = 16, ef_construction = 64);")


@dataclass(slots=True)
class Hit:
    chunk_id: str
    doc_id: str
    heading: str
    preview: str
    score: float


def upsert_chunks(rows: list[dict[str, Any]], dsn: str) -> int:
    """一个事务全写：ON CONFLICT DO UPDATE 保证增量重跑不产生副本。"""
    sql = """
        INSERT INTO chunks (chunk_id, doc_id, seq, heading, content, tokens, metadata, embedding)
        VALUES (%(cid)s, %(doc)s, %(seq)s, %(head)s, %(content)s, %(tok)s, %(meta)s, %(emb)s::vector)
        ON CONFLICT (chunk_id) DO UPDATE
        SET content = EXCLUDED.content, metadata = EXCLUDED.metadata,
            embedding = EXCLUDED.embedding, updated_at = now();
    """
    with psycopg.connect(dsn) as conn, conn.transaction():
        for r in rows:
            conn.execute(sql, {
                "cid": r["chunk_id"], "doc": r["doc_id"], "seq": r["seq"], "head": r["heading"],
                "content": r["text"], "tok": r["tokens"], "meta": json.dumps(r["metadata"]),
                "emb": "[" + ",".join(f"{x:.6f}" for x in r["embedding"]) + "]",  # 字面量就是方括号串
            })
    return len(rows)


def search(dsn: str, qvec: list[float], k: int = 5,
           docs: list[str] | None = None, max_sec: int = 3) -> list[Hit]:
    lit = "[" + ",".join(f"{x:.6f}" for x in qvec) + "]"
    sql = """
        SELECT chunk_id, doc_id, heading, left(content, 120),
               1 - (embedding <=> %(q)s::vector)
        FROM chunks
        WHERE (%(docs)s::text[] IS NULL OR doc_id = ANY(%(docs)s))
          AND coalesce((metadata->>'sec_level')::int, 0) <= %(max_sec)s
        ORDER BY embedding <=> %(q)s::vector LIMIT %(k)s;
    """
    with psycopg.connect(dsn) as conn:
        conn.execute("SET hnsw.ef_search = 100;")   # 会话级；服务里按请求需要设
        return [Hit(*row) for row in conn.execute(sql, {"q": lit, "docs": docs, "max_sec": max_sec, "k": k})]


if __name__ == "__main__":
    dsn = os.environ["PG_DSN"]   # postgresql://user:pw@localhost:5432/agentlab
    with psycopg.connect(dsn) as conn:
        conn.execute("CREATE EXTENSION IF NOT EXISTS vector;")
        conn.execute(DDL)
        conn.execute(HNSW)       # 数据入库后再建索引更快
    print("ready")
```

Qdrant 最小对照（够你决策，不必深入）：

```python
from qdrant_client import QdrantClient, models

qc = QdrantClient(url="http://localhost:6333")
qc.create_collection("chunks", vectors_config=models.VectorParams(size=1536, distance=models.Distance.COSINE))
qc.upsert("chunks", points=[models.PointStruct(id=r["chunk_id"], vector=r["embedding"], payload=r) for r in rows])
res = qc.query_points("chunks", query=qvec, limit=5, query_filter=models.Filter(
    must=[models.FieldCondition(key="doc_id", match=models.MatchValue(value="d1"))]))
# 结果读 .points[i].score / .payload；qdrant-client 迭代快，字段名以当前文档为准
```

**今天的动作**

- [ ] 本地起 PostgreSQL（Docker 一条命令），`CREATE EXTENSION vector` 成功。
- [ ] 建表并插 3 条手工数据，故意插 1 条错维度，记下报错原文。
- [ ] 建 HNSW，跑 `EXPLAIN ANALYZE` 确认走索引；把 `<=>` 换成 `<->` 再跑一次看索引失效。**这个对比是今天的核心收获。**
- [ ] 用一条真实问题查询，确认 score 与人工判断一致。
- [ ] `PG_DSN` 进 `.env`（并确保 `.gitignore` 覆盖），提交建表脚本。

**会踩的坑**

1. **查询算子与索引操作符类不一致**：结果仍正确只是变慢，最难发现。诊断：`EXPLAIN` 里看不到 `Index Scan using chunks_emb_idx`。
2. **表很小却抱怨 ANN 不准**：<1 万行建议不建索引，先量测再决定。
3. **过滤选择率极高导致返回不足 k 条或超时**：`SET hnsw.ef_search = 400` 复测，结果变全就是过滤与索引交互问题，不是 embedding 问题。
4. **向量存成 text/jsonb 自己解析**：入库能过，检索只能 Python 全表算距离，量一上来就死。

**自测**

1. `<=>` 返回 0.28 时余弦相似度是多少？归一化下 L2 距离是多少？
2. `probes` 与 `ef_search` 调大的效果和代价分别是什么？本周为什么不选 IVFFlat？
3. "元数据过滤能直接写进 WHERE" 这件事为什么对你后面的权限设计是决定性的？（W14 周五用到）

---

## 周六 · 深度块（120 分钟）：`build_index.py` + `search.py`

本周产出物是两个能反复跑的脚本：解析→清洗→切分→向量化→入库→检索必须串成一条命令，且**增量重跑不产生重复数据**。

### 第一段 55' · `build_index.py`

```python
"""build_index.py — 解析→清洗→切分→向量化→入库。
用法：python build_index.py [--doc ops-manual] [--dry-run] [--mode structured]
"""
from __future__ import annotations

import argparse
import json
import os
import time
from pathlib import Path
from typing import Any

import psycopg

from cleaning import drop_boilerplate, sanitize
from chunker import Chunk, build_chunks, chunk_hash
from emb_probe import embed
from vector_store import upsert_chunks

STATE = Path("data/index_state.json")


def load_chunks(path: Path, mode: str, size: int) -> list[Chunk]:
    pages = path.read_text(encoding="utf-8", errors="replace").split("\f")
    body = "\n\n".join(sanitize(p) for p in drop_boilerplate(pages) if p.strip())
    assert len(body) > 100, f"{path.name} 抽取为空，多半是扫描件"
    return build_chunks(body, path.stem, mode=mode, size=size, overlap=size // 8)


def doc_fingerprint(cs: list[Chunk]) -> str:
    """整篇指纹：与库里相同就跳过，省掉一次全篇 embedding 调用。"""
    return chunk_hash("|".join(c.chunk_id for c in cs))


def main() -> None:
    ap = argparse.ArgumentParser()
    ap.add_argument("--data", default="data/raw")
    ap.add_argument("--doc", help="只重建指定 doc_id")
    ap.add_argument("--mode", default="structured")
    ap.add_argument("--size", type=int, default=400)
    ap.add_argument("--dry-run", action="store_true")
    args = ap.parse_args()

    state: dict[str, str] = json.loads(STATE.read_text()) if STATE.exists() else {}
    todo: list[dict[str, Any]] = []
    skipped = 0
    for f in sorted(Path(args.data).glob("*")):
        if f.suffix not in {".md", ".html", ".txt"} or (args.doc and f.stem != args.doc):
            continue
        cs = load_chunks(f, args.mode, args.size)
        fp = doc_fingerprint(cs)
        if state.get(f.stem) == fp and not args.doc:
            skipped += 1
            continue
        state[f.stem] = fp
        todo += [{"chunk_id": c.chunk_id, "doc_id": c.doc_id, "seq": c.seq, "heading": c.heading,
                  "text": c.text, "tokens": c.tokens,
                  "metadata": {"sec_level": 1, "source": f.name, "fp": fp}} for c in cs]

    cost = sum(r["tokens"] for r in todo)
    print(f"跳过未变更 {skipped} 篇；待写 chunk {len(todo)}；待嵌 token {cost}")
    if args.dry_run:
        STATE.write_text(json.dumps(state, ensure_ascii=False, indent=2), encoding="utf-8")
        return

    t0 = time.perf_counter()
    if args.doc:   # 粗暴但正确的增量：整篇先删后插，避免旧内容哈希留在库里命中过期条款
        with psycopg.connect(os.environ["PG_DSN"]) as conn, conn.transaction():
            conn.execute("DELETE FROM chunks WHERE doc_id = %s", (args.doc,))
    for r, v in zip(todo, embed([r["text"] for r in todo])):
        r["embedding"] = v
    n = upsert_chunks(todo, os.environ["PG_DSN"])
    print(f"入库 {n} 条，耗时 {time.perf_counter() - t0:.1f}s")
    STATE.write_text(json.dumps(state, ensure_ascii=False, indent=2), encoding="utf-8")


if __name__ == "__main__":
    main()
```

增量语义要说清，它决定 W15 的 `reindex.py` 好不好写：`chunk_id` 是内容哈希 → 内容不变则主键冲突走 UPDATE 不产生副本；内容变了会产生新 id，**旧 id 必须删**，否则库里留着过期条款，检索命中旧版本。今天用"按 doc 先删后插"这个粗暴但正确的办法；真正的版本标记与保留历史是 W15 周五的事，今天别做。

### 休息 10'（离开屏幕，这段不省）

### 第二段 55' · `search.py` + 收口

```python
"""search.py — 命令行输入问题，返回 top-k 片段与分数。本周只做"能查"。"""
from __future__ import annotations

import argparse
import json
import os

from emb_probe import embed, normalize
from vector_store import Hit, search


def show(hits: list[Hit]) -> None:
    if not hits:
        # 先怀疑元数据过滤把候选滤空，再怀疑 embedding——顺序反了会白调一天
        print("无结果：检查 max_sec / doc 过滤是否过严")
        return
    for i, h in enumerate(hits, 1):
        print(f"[{i}] score={h.score:.4f} doc={h.doc_id} chunk={h.chunk_id}\n"
              f"    {h.heading or '-'}\n    {h.preview}\n")


def main() -> None:
    ap = argparse.ArgumentParser()
    ap.add_argument("query")
    ap.add_argument("-k", type=int, default=5)
    ap.add_argument("--doc", action="append")
    ap.add_argument("--max-sec", type=int, default=3)
    args = ap.parse_args()

    qv = normalize(embed([args.query])[0])
    hits = search(os.environ["PG_DSN"], qv, k=args.k, docs=args.doc, max_sec=args.max_sec)
    show(hits)
    with open("experiments/w13_search_log.jsonl", "a", encoding="utf-8") as f:
        f.write(json.dumps({"query": args.query, "k": args.k,
                            "hits": [{"chunk_id": h.chunk_id, "score": round(h.score, 6)} for h in hits]},
                           ensure_ascii=False) + "\n")


if __name__ == "__main__":
    main()
```

最后 15 分钟收口，不许省：

- [ ] `build_index.py --dry-run` 输出 chunk 数与预计 embedding token 量，写进日志。
- [ ] 全量建索引一次，记下 `SELECT count(*)` 与 `pg_size_pretty(pg_total_relation_size('chunks'))`。
- [ ] **连跑两次 `build_index.py`，第二次必须显示待写 0 且 `count(*)` 不变**——幂等验证不过就重做。
- [ ] `search.py` 跑 10 个真实问题，逐条人工判"命中/未命中/部分命中"，记进 `experiments/w13_manual_check.md`。**这 10 行判断就是 W14 周一标注集的前 10 条。**
- [ ] 更新 `learning-log.md` 三行记录 + 本周状态（完成 / 部分（缺什么）/ 未做），提交。

---

## 本周术语卡

| 术语 | 一句话解释 | 为什么重要 |
|---|---|---|
| 检索增强生成 RAG | 推理时从外部库捞片段注入上下文，知识不进模型参数 | 它是知识更新与可追溯的唯一低成本路径 |
| 索引期 / 查询期 | 离线建库与在线检索两个独立阶段 | 换 embedding 要重建索引期，影响面一眼看清 |
| 分块 Chunking | 把文档切成检索的原子单位 | 粒度决定 recall 上限，后续优化全在其上 |
| 重叠 Overlap | 相邻 chunk 之间重复的内容 | 防跨边界丢失，代价是副本膨胀与 top-k 占位 |
| 嵌入向量 Embedding | 文本的固定维稠密向量表示 | 语义相似被降维成可算距离，检索才有算法可用 |
| 余弦相似度 / 距离 | `sim = a·b/(|a||b|)`；距离 = 1 − sim | 阈值、索引算子、跨库比较全用它，换算错就全错 |
| 归一化 Normalization | 把向量模长缩为 1 | 归一化后 L2 与余弦排序等价，长文本不再不公平 |
| 近似最近邻 ANN | 用一点精度换 top-k 速度的索引搜索 | 上量后唯一可行路径，调参就是调 recall 与延迟 |
| HNSW / IVFFlat | 图索引 / 聚类分桶索引 | 增删频繁选 HNSW，静态大批量可选 IVFFlat |
| 操作符类 operator class | 索引绑定的距离算子（`vector_cosine_ops` 等） | 与查询算子不一致时索引静默不生效 |
| 模板噪声 Boilerplate | 跨页重复的页眉页脚水印 | 系统性重复会拉近所有 chunk 向量，抹平区分度 |
| 元数据过滤 Metadata filter | 检索前用属性条件缩小候选集 | 权限与数值判断只能靠它，embedding 靠不住 |

## 延伸阅读（可选，通勤时段）

pgvector 官方 README 只读两节：Indexes（HNSW/IVFFlat 参数与前提）、Distance metrics（三个算子语义）。你项目所用 embedding 服务商文档里的"维度 / 批上限 / 速率限制"三节。其余一律不看。
