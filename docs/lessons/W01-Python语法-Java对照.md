# W01 · Python 语法速通（面向 Java 开发者）

> 本周是四周地基的第一周。结束时你要拥有一个干净的 Python 3.12 环境、一个组织成包的小工具库，以及一张自己踩坑踩出来的 Java→Python 对照表。这周不碰大模型，别急。

## 先看你本机现在的情况

这些是我实际探测过的结果，不是猜测：

| 项 | 现状 | 结论 |
|---|---|---|
| `python3` | `/usr/bin/python3`，**Python 3.9.6**（Xcode 命令行工具自带） | 必须装 3.12+。LangGraph 等 Agent 库要求 3.10+，`int \| None` 这类联合类型语法也需要 3.10+。系统 Python 更不该被污染——它是 Xcode 的依赖 |
| `pip` | 21.2.4，指向 CLT 的 site-packages | 先别用。装包会撞上系统目录写权限问题 |
| `uv` / `pipx` | **未安装** | 周一第一件事就是装它 |
| `java` | **1.8.0_441** | 现在不影响你。但 W23 要跑 Spring AI 2.0 demo 需要 JDK 21+，到那周再装，别提前动你的构建环境 |
| `docker` | 未安装 | W19（执行隔离）和 W22（容器化）之前装 OrbStack 或 Docker Desktop。现在不用管 |
| `git` | 2.49.0，身份已配（`lt / liteng@dcps.top`），`credential.helper=osxkeychain` | 可以直接用 |
| `gh` | 我正在给你装 | 装完仓库就能推到 GitHub |

## 本周导航

| 日 | 主题 | 今天的产出物 |
|---|---|---|
| 周一 | 环境与解释器 | Python 3.12 + uv + 虚拟环境跑通 `hello.py` |
| 周二 | 容器类型与推导式 | 一个用推导式重写你熟悉 Java Stream 逻辑的脚本 |
| 周三 | 函数是一等公民 | 3 个纯函数工具 + 一段闭包陷阱实验 |
| 周四 | 类、dataclass、Protocol | 一个 dataclass 模型 + Protocol 约束的工具接口 |
| 周五 | 异常与上下文管理器 | 自定义异常层级 + 一个自写的 `with` 资源类 |
| 周六 | 模块包结构与类型检查 | `python-for-java` 包 + pyright 全绿 + 首次 commit + 对照表 |
| 周日 | 休息，不许补课 | — |

---

## 周一 · 环境与解释器

**本质**：你需要一个和项目绑定的解释器与依赖集合。Python 没有 Maven 那种"全局仓库 + pom 精确解析"的心智模型，它是"每个虚拟环境一套 site-packages"，所以隔离从第一天就要当成习惯，不是等出问题再补。

**Java 视角**：`~/.m2/repository` 是全局缓存，所有项目共享；`venv` 更像每个项目自带一份 `target/dependency`，激活之后 `python` 这个命令本身就换成了项目里的那个。你不需要"全局装包"这个动作——绝大多数时候在系统 Python 里 `pip install` 都是错的。

**核心讲解**

*为什么必须换掉 3.9.6*：三个层面。一是库门槛，LangGraph、新版 OpenAI SDK 都要求 3.10+；二是语言特性，`int | None` 写法（PEP 604）、`match` 语句、更严格的类型检查都是 3.10+；三是它归 Xcode 管，你 `pip install` 可能破坏工具链，也可能被系统权限拒绝。

*选 uv 而不是 pip + venv + pyenv 三件套*：`uv` 一个工具同时管 Python 版本、虚拟环境、依赖解析和锁文件，快且行为可预测。类比一下它是 SDKMAN + Maven 的合体。你在企业里可能遇到 Poetry 或 PDM，概念一致，不必现在学。

*激活 vs 不激活*：所有 `uv run xxx` 形式的命令都不需要你手动激活环境，它自己找。手动 `source .venv/bin/activate` 只是为了让当前终端里的 `python` 指向项目解释器。两种都行的时候优先 `uv run`，少一层状态。

**代码**

```bash
# 1. 装 uv（Homebrew 你已经有：/opt/homebrew/bin/brew）
brew install uv

# 2. 装 Python 3.12 到 uv 自己管理的位置（不碰系统 Python）
uv python install 3.12

# 3. 进你的学习仓库目录，初始化一个绑 3.12 的项目
cd ~/Desktop/AI/agent-lab
uv init --name agent-lab --python 3.12   # 生成 pyproject.toml 与 .python-version
uv sync                                   # 建 .venv 并按 pyproject 装依赖

# 4. 验证：解释器版本与路径都应该是 3.12 和 ./.venv 下
uv run python -c "import sys; print(sys.version); print(sys.executable)"
```

```python
# hello.py —— 就三行，目的是确认工具链闭环：编辑器用的解释器 == 你运行的解释器
from __future__ import annotations

print("agent-lab", __import__("sys").version.split()[0])
```

编辑器侧：VS Code 装 Python 扩展后 `Cmd+Shift+P` → "Python: Select Interpreter" → 选 `.venv/bin/python` 那个。PyCharm 则 Settings → Project → Python Interpreter → Add Local → 选已有 `venv`，路径填 `.venv`。**这一步不确认，后面每次 import 报红的排查都是白花时间。**

**今天的动作**

- [ ] `brew install uv` 成功，`uv --version` 有输出。
- [ ] `uv python install 3.12` 完成，`uv python list` 里能看到 3.12。
- [ ] `agent-lab` 目录下有 `pyproject.toml`、`.python-version`、`.venv/`。
- [ ] `uv run python hello.py` 打印出 3.12.x。
- [ ] 编辑器右下角显示的解释器路径指向 `agent-lab/.venv`。
- [ ] `git add -A && git commit -m "chore: python 3.12 + uv 环境初始化"`（`.venv` 必须先进 `.gitignore`）。

**会踩的坑**

1. `python` 命令找不到或仍是 3.9。原因：macOS 默认只有 `python3`，且 shell 有 hash 缓存。解决：一律用 `uv run python`，或先 `source .venv/bin/activate` 再 `hash -r`。
2. 编辑器选错解释器，写 `import httpx` 报"未解析"，命令行却能跑。诊断：看编辑器右下角那个版本号——它和 `uv run python -c "import sys; print(sys.executable)"` 不一致就是它。
3. 把 `.venv/` 提交进 git。体积几十 MB 且必冲突。`.gitignore` 第一行就该有它。
4. 顺手 `sudo pip install`。立刻停手，你的环境现在有两个来源，之后每次 import 冲突都难查。

**自测**

1. 你的项目里 `pip install` 实际把包装到哪个绝对路径？怎么验证？
2. `uv run python x.py` 和不激活环境直接 `python3 x.py`，行为差别是什么？为什么？
3. `.python-version` 和 `pyproject.toml` 里 `requires-python` 各自的作用是什么？

---

## 周二 · 容器类型与推导式

**本质**：Python 的 list / dict / set / tuple 是带字面量语法的内建类型，不需要构造器；推导式是"用集合本身的写法描述集合的生成过程"，读起来比 Stream 链更紧凑，但表达力上限也更低。

**Java 视角**：`Stream.map().filter().collect(groupingBy())` 那一整套，Python 里前两步用推导式、分组用 `defaultdict` 或 `itertools.groupby`（后者有坑，见下）。Python 没有 `List.of` 与 `Arrays.asList` 的固定/可变之分——所有 list 都可变，除非你自己用 tuple。

**核心讲解**

*四种容器的语义边界*：list 有序可变、tuple 有序不可变（但元素本身可以是可变对象）、dict 有序（3.7+ 保证插入序）可变、set 无序去重。能被放进 set / 当 dict key 的对象必须可哈希，即值不可变——这就是为什么 `dict.fromkeys([])` 会报错而 `dict.fromkeys(())` 可以。

*推导式的边界*：只能做 map 和 filter，做不了带副作用的循环、做不了多趟聚合。需要聚合就用显式循环或标准库，别硬塞。嵌套推导式超过两层就改成循环，可读性优先——这条比"用推导式显得 Pythonic"重要得多。

*dict 的三个高频方法*：`get(k, default)` 避免 KeyError；`setdefault(k, default)` 在 key 不存在时写入并返回；`items()` 遍历键值。这三者能替掉你 Java 里 `containsKey` + `get` 的双查。

**代码**

```python
# 场景：Java 里你写过 list.stream().filter(...).map(...).collect(groupingBy(...))
from collections import Counter, defaultdict
from dataclasses import dataclass


@dataclass
class Order:
    id: str
    user: str
    amount: float
    status: str


orders = [
    Order("o1", "u1", 120.0, "paid"),
    Order("o2", "u2", 30.0, "pending"),
    Order("o3", "u1", 75.5, "paid"),
    Order("o4", "u1", 5.0, "refunded"),
]

# 过滤 + 映射：一行，等价于 filter().map().toList()
paid_amounts = [o.amount for o in orders if o.status == "paid"]

# 分组：defaultdict 比 groupby 更好用（groupby 要求数据已按 key 排序，反直觉）
by_user: dict[str, list[Order]] = defaultdict(list)
for o in orders:
    by_user[o.user].append(o)

# 字典推导 + 聚合
user_total = {u: sum(o.amount for o in vs) for u, vs in by_user.items()}

# 计数
status_count = Counter(o.status for o in orders)  # Counter({'paid': 2, ...})

# any/all 短路，替代 Java 的 anyMatch/allMatch
has_big = any(o.amount > 100 for o in orders)
all_valid = all(o.amount > 0 for o in orders)

# 解包与枚举：Python 里元组解构无处不在，后面处理 API 响应会大量用到
first, *rest = paid_amounts          # rest 永远是 list，哪怕空
for idx, o in enumerate(orders, start=1):
    pass                             # 别写 range(len(orders))

# 生成器表达式：把中括号换成小括号就是惰性求值，不建中间 list
total = sum(o.amount for o in orders)
```

**今天的动作**

- [ ] 找出你最近工作项目里一段 Java Stream 处理（过滤 + 分组 + 聚合都最好有）。
- [ ] 用 Python 重写它，不许用 pandas，只许推导式 + `defaultdict` + `Counter`。
- [ ] 再加一个嵌套 dict 推导式，输出"每个用户每种状态的数量"。
- [ ] 用 `enumerate`、解包、`any/all` 各改写一处原来的写法。
- [ ] 提交，commit message 写清今天重写的是哪个业务逻辑。

**会踩的坑**

1. **可变默认参数**：`def f(items=[])` 里的 `[]` 在函数定义时创建一次，所有调用共享。这是 Java 里没有的 bug 类型。写 `items: list | None = None` 然后函数内 `items = items or []`。
2. `itertools.groupby` 不排序，只压缩相邻相同 key。数据没按 key 排过序，结果会莫名其妙分裂成多组。要分组就 `defaultdict` / `dict.setdefault`。
3. 浅拷贝陷阱：`b = a` 不复制、`b = a[:]` 和 `list(a)` 都只复制一层，元素仍是共享对象。改嵌套结构需要 `copy.deepcopy`（慢，先想想是不是设计有问题）。
4. `==` 比值、`is` 比对象身份。小整数和字符串有驻留，让你误以为 `is` 能比内容。判断字面量（`None`/`True`/`False`）才用 `is`，其余一律 `==`。

**自测**

1. `dict.fromkeys([("a", 1)])` 为什么报错，而元组版本可以？
2. `defaultdict(list)` 和 `d.setdefault(k, []).append(v)` 哪个更快，为什么？
3. 写出 `*rest` 解包时 Python 对剩余元素做了什么，空剩余时 `rest` 是什么。

---

## 周三 · 函数是一等公民

**本质**：函数是对象，能被存进 dict、当参数传、被返回、被包装。Python 用参数协议（默认值、关键字、可变参数）取代了 Java 的重载。

**Java 视角**：没有方法重载，同名函数后定义覆盖前定义；`Function<T,R>` / `@FunctionalInterface` 对应的就是裸函数 + `typing.Callable[[int], str]` 这个类型注解；`this` 不隐藏，函数就是函数。

**核心讲解**

*参数匹配的优先级*：位置参数 → 可变位置 `*args` → 仅限关键字（`*` 之后）→ 关键字参数 → `**kwargs`。写 Agent 工具层时你会大量用"仅限关键字"来强迫调用方显式写 `timeout=30` 而不是靠位置，这是可读性设计手段。

*lambda 只是语法糖*：只能包含单个表达式，不能赋值、不能 try、不能 raise。稍复杂就 `def`。它的正当用途只有一处：作为一次性回调传给 `sorted(key=...)`、`map` 等。

*装饰器*：`@foo` 等价于 `func = foo(func)`。它不神秘，就是高阶函数。W09 落 JSONL 日志、W21 接 Langfuse 追踪，都会写成装饰器。这周先练手写一个最简单的计时装饰器建立肌肉记忆。

*闭包捕获的是变量，不是值*：这是本周唯一真正的智力难点，也是 Agent 代码里循环注册工具时最常见的 bug。

**代码**

```python
import functools
import time
from typing import Any, Callable


def timed(func: Callable[..., Any]) -> Callable[..., Any]:
    """最小装饰器：打印耗时。保留元信息必须用 functools.wraps，否则函数名变成 'wrapper'"""

    @functools.wraps(func)
    def wrapper(*args: Any, **kwargs: Any) -> Any:
        start = time.perf_counter()
        try:
            return func(*args, **kwargs)
        finally:
            print(f"{func.__name__} 耗时 {time.perf_counter() - start:.3f}s")

    return wrapper


@timed
def slow_add(a: int, b: int) -> int:
    time.sleep(0.2)
    return a + b


# 仅限关键字参数：强迫调用方写出 timeout= 与 retries=，可读性优先
def http_get(url: str, *, timeout: float = 5.0, retries: int = 2) -> str:
    return url * retries


def make_reporter(name: str) -> Callable[[], None]:
    def report() -> None:
        print(f"[{name}] reporting")

    return report


# 闭包陷阱：循环里创建的函数捕获的是变量本身，循环结束时全部指向最后一个值
handlers_wrong = [lambda: print(i) for i in range(3)]     # 三次都打印 2
handlers_right = [lambda i=i: print(i) for i in range(3)]  # 用默认参数固化当前值
# 更清楚的写法：
handlers_clear = [make_reporter(f"worker-{i}") for i in range(3)]
```

**今天的动作**

- [ ] 写 `timed` 装饰器并装饰一个 `time.sleep` 函数，观察 `functools.wraps` 去掉前后 `.__name__` 的变化。
- [ ] 写一个 `def summarize(items, *, top_k: int = 5, min_len: int = 2)`，用仅限关键字参数，然后故意用位置传参看报错信息。
- [ ] 复现上面的闭包陷阱，用 `python -c` 跑一次，亲眼看到三次打印 `2`。
- [ ] 用 `functools.partial` 把一个 3 参数函数固定成 1 参数函数，塞进 dict 当策略表：`{"fast": fn, "slow": partial(fn, timeout=30)}`。
- [ ] 提交。

**会踩的坑**

1. 忘写 `functools.wraps`，被装饰函数的 `__name__`、`__doc__` 丢失。Agent 工具注册靠这些名字生成 schema 时，你会拿到一堆叫 `wrapper` 的工具——模型完全不知道该调哪个。
2. 闭包捕获循环变量（上面已给三种解法，默认参数那招要理解而不是背）。
3. `nonlocal` 与 `global`：内层函数对外层变量赋值会被当成创建局部变量，需要显式声明。读出来报错说 "referenced before assignment" 就是它。
4. 可变参数 `**kwargs` 一路透传到底，最终没有任何地方能校验参数拼错。库边界要收口成显式签名。

**自测**

1. `*` 单独出现在参数列表里意味着什么？为什么 Agent 工具函数应该用它？
2. `lambda` 里能不能写 `try/except`？不能的话正确做法是什么？
3. 装饰器的执行时机是定义时还是调用时？用一个打印语句实验证明。

---

## 周四 · 类、dataclass、Protocol

**本质**：Python 的类主要用来组织数据和行为。类型系统靠鸭子类型——不看继承关系，只看有没有对应方法。

**Java 视角**：`record` ≈ `@dataclass(frozen=True, slots=True)`；`interface` + `implements` ≈ `Protocol`，但区别在于 Protocol 是**结构化子类型**，你的类不需要显式声明实现了它，静态检查看方法签名匹配就认——这既是自由也是风险，因为不满足协议时只有 mypy/pyright 会报，运行时不报错。没有 `public/private`，靠 `_` 前缀约定；没有 `final`。

**核心讲解**

*为什么 dataclass 而不是裸 dict*：Agent 代码里到处是消息、工具调用、评测样本这类结构。裸 dict 拼错 key 静默返回 None，dataclass 拼错属性立刻 AttributeError。W02 学 pydantic 之后你会用 pydantic 处理外部输入，dataclass 处理内部对象——这个分工是标准做法：**边界用 pydantic 校验，内部用 dataclass 表达**。

*dunder 方法决定对象能不能被正确使用*：`__repr__`（调试可读性）、`__eq__`/`__hash__`（能否进 set/dict）、`__len__`/`__iter__`（能否 for 循环和解包）、`__enter__`/`__exit__`（能否 with）。dataclass 自动给了前两个。

*Protocol 用在"我要求任何工具都长这样"*：这是你设计 Agent 工具层时的核心抽象，比继承一个 BaseTool 更灵活。

**代码**

```python
from dataclasses import dataclass, field
from typing import Protocol, runtime_checkable


@dataclass(slots=True)
class ToolCall:
    """模型请求的一次工具调用。内部对象用 dataclass，字段顺序即展示顺序。"""

    id: str
    name: str
    arguments: str  # 原始 JSON 字符串，解析失败要能追溯到原文
    parsed: dict = field(default_factory=dict)


@dataclass(frozen=True, slots=True)
class Money:
    amount_cents: int
    currency: str = "CNY"

    def __add__(self, other: "Money") -> "Money":
        if self.currency != other.currency:
            raise ValueError(f"不能混算币种: {self.currency} vs {other.currency}")
        return Money(self.amount_cents + other.amount_cents, self.currency)


@runtime_checkable
class Tool(Protocol):
    """工具协议：Agent 的工具层只依赖这个形状，不依赖任何基类。"""

    name: str
    description: str

    def run(self, **kwargs: object) -> str: ...


@dataclass(slots=True)
class SqlQueryTool:
    name: str = "sql_query"
    description: str = "对只读副本执行 SQL 查询，返回最多 200 行 JSON"

    def run(self, **kwargs: object) -> str:
        return f"executed: {kwargs}"


def register(tools: list[Tool]) -> dict[str, Tool]:
    """依赖 Protocol 而不是具体类：任何满足形状的对象都能注册。"""
    return {t.name: t for t in tools}
```

**今天的动作**

- [ ] 用 `@dataclass(slots=True)` 建一个 `ChatMessage`（role / content / name 可选 / tool_calls 列表默认空）。
- [ ] 加一个 `@dataclass(frozen=True)` 的 `EvalSample`（question / expected_doc_ids / should_refuse），这是 W21 评测集的雏形。
- [ ] 写一个 `Tool` Protocol 和两个实现，用 `register()` 收进 dict。
- [ ] 故意写一个不满足 Protocol 的类传进去，看 pyright 报不报错、运行时报不报错——**这个实验让你真正理解结构化子类型**。
- [ ] 提交。

**会踩的坑**

1. `@dataclass` 里写 `tags: list[str] = []` 直接报错，必须 `field(default_factory=list)`。原因同周二的可变默认参数：默认值在类定义时求值一次。
2. 忘写 `self` 参数，或写了 `self` 却在外部当普通函数调用。
3. 定义了 `__eq__` 却没定义 `__hash__`，对象自动变成不可哈希，进不了 set。frozen dataclass 会自动处理，非 frozen 会给你 `eq=True, frozen=False` → 不可哈希。
4. 用继承搭深度类体系（Java 习惯）。Python 里优先组合 + Protocol + 少量 mixin；`super()` 在多继承下的 MRO 行为和你直觉不同，不要在没有明确需要时多继承。

**自测**

1. `slots=True` 省了什么、代价是什么？
2. Protocol 与 ABC 的差别，什么时候必须用 ABC？
3. 为什么运行时不满足 Protocol 不会抛异常？这对你写工具注册校验有什么影响？

---

## 周五 · 异常与上下文管理器

**本质**：Python 没有 checked exception，异常靠约定和测试兜住；`with` 把"获取/释放"这对动作变成协议，资源泄漏从此不依赖你记得写 finally。

**Java 视角**：`try-with-resources` ≈ `with`（`__enter__` / `__exit__`）；`throws` 声明不存在，编译器不强制处理——**代价是运行时的错误覆盖面只能靠你自己设计异常层级 + 测试保证**，这点你比纯算法背景的人更有感觉。`catch (A | B e)` ≈ `except (A, B) as e`。

**核心讲解**

*结构是 `try / except / else / finally`*：`else` 只在没有异常时执行，用来把"正常路径"从 try 块里挪出来，避免误捕 else 里的异常——这个语义 Java 里没有对应物。

*EAFP 而非 LBYL*：Python 习惯"先做，出错再处理"（try 里调 `d[key]`），而不是"先检查再做"（`if key in d`）。因为前者在并发下不脆弱、且少一次查找。异常便宜、正常路径才贵，这与 Java 的性能直觉相反。

*重新抛出要带链*：`raise NewError(...) from exc` 保留 `__cause__`，日志里能追到原始异常。裸 `raise NewError(exc)` 会给你一条更弱的链。

*`with` 的实现*：`__enter__` 返回值给 `as`，`__exit__(exc_type, exc, tb)` 返回 `True` 会**吞掉异常**——除非你确定，否则返回 None。

**代码**

```python
import time
from dataclasses import dataclass
from pathlib import Path


class AgentError(Exception):
    """项目异常基类：上层只 catch 这个，就能捕获全部业务异常。"""


class ToolTimeout(AgentError):
    def __init__(self, tool: str, seconds: float) -> None:
        super().__init__(f"工具 {tool} 超时 {seconds}s")
        self.tool = tool
        self.seconds = seconds


class ConfigMissing(AgentError):
    """注意：不继承 KeyError —— 语义不同，继承结构要为排查服务。"""


def load_prompt(name: str, prompts_dir: Path) -> str:
    path = prompts_dir / f"{name}.md"
    try:
        return path.read_text(encoding="utf-8")
    except FileNotFoundError as exc:
        raise ConfigMissing(f"提示词文件不存在: {path}") from exc
    # 注意这里不 except OSError 以外的东西：编程错误必须冒出来


@dataclass(slots=True)
class Timer:
    label: str

    def __enter__(self) -> "Timer":
        self.start = time.perf_counter()
        return self

    def __exit__(self, exc_type, exc, tb) -> None:  # 返回 None：绝不吞异常
        print(f"[{self.label}] {time.perf_counter() - self.start:.3f}s")
        # 如果 exc_type is not None，异常会继续向上抛，这里只是记录


with Timer("检索"):
    time.sleep(0.1)
```

**今天的动作**

- [ ] 建 `src/exceptions.py`：`AgentError` 基类 + 3 个子类（工具超时 / 工具失败 / 配置缺失），每个带结构化字段（不是只有 message）。
- [ ] 把周三的工具函数包一层：失败时抛你自定义的异常而不是返回 `None` 或空串。
- [ ] 自己实现一个 `Timer` 上下文管理器（如上），再实现一个"写入临时文件、异常时自动删除"的上下文管理器。
- [ ] 实验：`__exit__` 返回 `True` 时异常会不会消失？亲眼验证一次并在笔记里写下后果。
- [ ] 提交。

**会踩的坑**

1. `except Exception:` 一把抓会吞掉 `TypeError`、`AttributeError` 这类纯编程错误，让 bug 变成"今天模型没答上来"。捕获范围收到最小：只捕你会造成的那些。
2. 裸 `except:` 连 `KeyboardInterrupt` 都吞，Ctrl+C 停不掉进程。永远写具体类型。
3. 把 API 错误当"空结果"返回（`except: return []`）。Agent 拿到空结果会认为"确实没有数据"，然后一本正经地胡说。**失败必须显式传给模型**，这点 W06 会展开。
4. `finally` 里 `return` 会吞掉 try 里的异常和返回值，几乎总是 bug。

**自测**

1. `else` 块和把代码写在 try 之后，差别在哪？举一个会因此漏捕/误捕的例子。
2. `raise X from Y` 与 `raise X(Y)` 在 traceback 上的区别？
3. 为什么自定义异常要带结构化字段？在 Agent 场景里它怎么被用上？

---

## 周六 · 模块包结构、类型注解、静态检查（120 分钟）

**本质**：模块就是一个 `.py` 文件，包是一个目录。导入是**运行时**按 `sys.path` 查找并**执行**该文件的顶层代码——这两点解释了 90% 的 import 报错。

**Java 视角**：没有编译期 classpath 与"package 声明必须匹配目录"的强制，全靠约定和 `sys.path`；`import a.b.C` ≈ `import a.b.C`，但没有同名类冲突解决机制（靠命名空间/别名）；`if __name__ == "__main__"` 的作用类似"只有被直接执行时才跑 main"，等价于你必须在 Java 里给每个类单独写 main 的限制的反面。

**核心讲解**

*导入执行顶层代码*：import 一个模块就等于运行它。所以模块顶层不要放重活、不要放副作用（连数据库、读环境变量后立刻打印）。这是"改了 A.py 结果 B.py 行为变了"的根因。

*`python x.py` 与 `python -m pkg.mod` 的区别*：后者把包上下文建立起来，相对导入才可用，且 `sys.path[0]` 是当前目录。项目一变大之后你一定会撞上 "attempted relative import with no known parent package"，答案就是改用 `-m`。

*类型注解不参与运行*：注解默认不校验、不影响行为，它是给你和 pyright 看的。真实运行时校验交给 W02 的 pydantic。别指望 `x: int = "a"` 报错。

*静态检查现在就要配*：Python 没有 javac 这道门，全靠 pyright/mypy 补。Java 工程师最大的适应成本是"错误在运行时才暴露"——把静态检查接进编辑器之后这个痛感会降一半。

**代码**

```toml
# pyproject.toml —— 相当于 pom.xml，但只管声明依赖与工具配置
[project]
name = "agent-lab"
version = "0.1.0"
requires-python = ">=3.12"
dependencies = []          # 本周留空，W02 再加 httpx / pydantic

[tool.pyright]
include = ["src", "tests"]
typeCheckingMode = "standard"   # 严格模式太吵，standard 够用；想升级用 "strict"
reportMissingImports = "error"
```

```python
# src/pyforjava/strings.py
"""本周工具函数集中地。W02 起会被 src/llm/ 引用，所以结构现在就定好。"""


def group_by(items, key_fn):
    """返回 dict[key, list[item]]，用 dict 而 defaultdict：对外更明确。"""
    out: dict[object, list] = {}
    for it in items:
        out.setdefault(key_fn(it), []).append(it)
    return out


def chunked(seq: list, size: int):
    """W13 切分文档、W17 并发分批都会直接复用这个函数，现在就写测试。"""
    if size <= 0:
        raise ValueError("size 必须为正")
    return [seq[i : i + size] for i in range(0, len(seq), size)]
```

```bash
# 静态检查与格式化，一次性配好
uv add --dev pyright ruff
uv run pyright            # 看 src/ 有没有报错
uv run ruff check --fix . # 替代你手动整理 import
```

**今天的动作（分两段，中间休息 10 分钟）**

第一段 55'：
- [ ] 把本周脚本全部移进 `src/pyforjava/`，加 `__init__.py`，删掉所有相互依赖的临时文件。
- [ ] 配 `pyproject.toml` + pyright，跑到零报错。
- [ ] 给 `group_by` 和 `chunked` 各写 3 条 `assert` 冒烟测试（正式 pytest 在 W02）。

第二段 55'：
- [ ] 写 `docs/java-to-python.md`：只写你**真的踩过的**差异，每条一句"表现 + 原因"。目标 12 条以上。参考条目：可变默认参数、闭包捕获变量、没有重载、Protocol 是结构匹配、异常无 checked、`is` vs `==`、浅拷贝、`__init__.py` 会被执行、frozen dataclass 的哈希、`else` 块、`slots`、注解不校验。
- [ ] README 写清"这是什么、怎么跑起来、目录结构"。
- [ ] `git commit` 并把对照表链接更新到 `learning-log.md`。

最后 10'：填本周状态（完成 / 部分（缺什么）/ 未做），写一句"下周第一天要做什么"。

**会踩的坑**

1. 循环导入：A import B、B import A。解法是把共同依赖抽到第三个模块，或用 `if TYPE_CHECKING:` 只为注解导入。**后者是标准手法**：

```python
from typing import TYPE_CHECKING

if TYPE_CHECKING:          # 只在静态检查时可见，运行时不导入，斩断环
    from src.agent.loop import AgentLoop


def use(loop: "AgentLoop") -> None: ...
```

2. 自己建了 `json.py` / `types.py` / `email.py` 之类与标准库同名文件，之后 import 标准库拿到的是你自己的空文件。报错形态极其诡异（"module 'json' has no attribute 'loads'"）。立刻改名。
3. `sys.path` 里塞 `..` 来 hack 导入路径。这是把问题藏起来。用包结构 + `-m` 执行。

**自测**

1. `import a.b.c` 时 Python 按什么顺序找？找到之后做了什么？
2. `python -m src.pyforjava.strings` 和 `python src/pyforjava/strings.py` 差别是什么？
3. `if TYPE_CHECKING` 为什么不会带来运行时开销？

---

## 本周术语卡

| 术语 | 一句话解释 | 为什么重要 |
|---|---|---|
| 虚拟环境 venv | 一个项目独占的解释器 + site-packages 目录 | 避免"我能跑你不能跑"；Agent 库版本冲突极常见 |
| uv | Rust 写的 Python 版本/环境/依赖一体管理器 | 替代 pyenv+venv+pip+poetry 四件套 |
| 推导式 | 用字面量语法生成 list/dict/set | 替代大部分 Stream 链，但表达力有上限 |
| 可哈希 hashable | 值不可变、能进 set / 当 dict key | 决定数据结构能不能用你的对象 |
| 可变默认参数 | 默认值在 def 时求值一次并被共享 | Java 没有的 bug 类型，务必条件反射般避开 |
| 闭包 closure | 内层函数引用外层作用域变量 | 循环注册工具时的头号陷阱 |
| 装饰器 decorator | 接收函数返回函数，`@` 语法 | 计时、日志、追踪都靠它，W09/W21 大量用 |
| dataclass | 自动生成 `__init__`/`__repr__`/`__eq__` 的类 | 内部对象的默认选择；边界对象用 pydantic |
| Protocol | 结构化子类型：只看方法签名不看继承 | 工具层抽象的基础，不需要 implements |
| `__enter__` / `__exit__` | with 语句的两个钩子 | 资源释放协议；`__exit__` 返回 True 会吞异常 |
| EAFP | 先做再捕获，而非先检查再做 | Python 的异常经济模型，和 Java 直觉相反 |
| sys.path | 运行时模块搜索路径列表 | 一切 import 报错的排查起点 |
| pyright | 静态类型检查器 | 补上"没有编译期"这块短板 |
| TYPE_CHECKING | 只为静态检查存在的导入块 | 解循环导入的标准做法 |

## 延伸阅读（不占用你 1 小时额度，通勤时看）

- Python 官方 Tutorial 第 3–9 章（你有 Java 基础，读它比看视频快）。
- `pyproject.toml` 打包规范页（PEP 621）——只需知道自己那几行配置的含义。
