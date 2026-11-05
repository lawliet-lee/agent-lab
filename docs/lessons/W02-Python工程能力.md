# W02 · Python 工程能力（真实调用）

> 本周把"会写 Python"升级成"能写给别人跑的 Python"：HTTP 客户端、类型化边界、并发、配置、测试。周六你要拿到自己的第一个大模型 API key，并跑通第一条 chat completion。这是四周地基里唯一接触真模型的一周，但**本周刻意不装任何框架**。

## 本周导航

| 日 | 主题 | 今天的产出物 |
|---|---|---|
| 周一 | httpx 与真实 HTTP | 带超时/重试的 GET+POST 客户端脚本 |
| 周二 | pydantic v2 类型化边界 | 把 API 响应解析成可校验对象 |
| 周三 | asyncio 与并发 | 同步 vs 异步耗时对比数据 |
| 周四 | 配置与密钥 | `.env` + 统一 settings 模块，仓库零密钥 |
| 周五 | pytest | 5 个通过的测试 + 一个 fixture |
| 周六 | 第一个大模型调用 | `agent-lab` 主结构定型 + 一次真实对话与用量记录 |
| 周日 | 休息，不许补课 | — |

---

## 周一 · httpx 与真实 HTTP

**本质**：LLM 就是一次 HTTPS 请求。把这层彻底搞清楚，比先学一个封装好的 SDK 有价值得多——因为 Agent 出问题的时候，你需要知道线上到底发了什么。

**Java 视角**：`httpx` ≈ OkHttp / HttpClient，`requests` ≈ 更老的 Apache HttpClient。差别在 `httpx` 同时提供同步和异步两套接口，而 `requests` 只有同步——**所以本课程一律用 `httpx`，别学 `requests`**（它仍会在别人代码里出现，能看懂就行）。

**核心讲解**

*超时必须显式设置*。默认 `timeout=5.0` 看起来安全，但 LLM 响应经常 30 秒起步。超时是三段的：连接、读、写。读超时要按"最长 token 间隔"设，不是总时长——流式响应下两次 chunk 之间才计 read timeout。

*异常层次是排查依据*：`HTTPError` 是所有 HTTP 相关异常的基类；`raise_for_status()` 把 4xx/5xx 变成异常，你不写它就静默通过——Java 的 HttpClient 同样不自动抛，这个习惯要带过来。状态码 429 是限流，5xx 是服务端，4xx 除 429 外基本是你的请求体错了。

*重试的边界*：只对连接失败、超时、429、5xx 重试；对 400/401/403/422 重试毫无意义（模型请求 400 通常是你的 schema 错了）。指数退避 + 抖动，别做固定间隔——多进程同时撞限流会形成同步风暴。

**代码**

```python
import random
import time

import httpx


class HttpError(RuntimeError):
    def __init__(self, status: int | None, detail: str) -> None:
        super().__init__(f"[{status}] {detail}")
        self.status = status  # None 表示没拿到响应（连接失败/超时）


def request_with_retry(
    client: httpx.Client,
    method: str,
    url: str,
    *,
    max_retries: int = 3,
    base_delay: float = 0.5,
    **kwargs: object,
) -> httpx.Response:
    """指数退避 + 抖动。只对可重试错误重试，其余立即抛出。"""
    retryable_status = {429, 500, 502, 503, 504}
    last: Exception | None = None

    for attempt in range(max_retries + 1):
        try:
            resp = client.request(method, url, **kwargs)
            if resp.status_code in retryable_status and attempt < max_retries:
                raise HttpError(resp.status_code, resp.text[:200])
            resp.raise_for_status()
            return resp
        except (httpx.TimeoutException, httpx.ConnectError, HttpError) as exc:
            last = exc
            if attempt == max_retries:
                break
            status = getattr(exc, "status", None)
            if status is not None and 400 <= status < 500 and status != 429:
                raise  # 客户端错误重试无意义，直接暴露
            delay = base_delay * (2**attempt) + random.uniform(0, 0.3)
            time.sleep(delay)

    raise HttpError(None, f"重试 {max_retries} 次仍失败: {last}")


with httpx.Client(
    base_url="https://httpbin.org",
    timeout=httpx.Timeout(connect=5.0, read=30.0, write=10.0, pool=5.0),
    headers={"User-Agent": "agent-lab/0.1"},
) as client:  # with：连接池复用，等价于你复用 OkHttp 的 OkHttpClient 实例
    r = request_with_retry(client, "GET", "/get", params={"q": "python"})
    print(r.status_code, r.json()["args"])

    r2 = request_with_retry(client, "POST", "/post", json={"a": 1})
    print(r2.json()["json"])
```

**今天的动作**

- [ ] 装上 `httpx`（`uv add httpx`），把上面代码跑通。
- [ ] 故意把 URL 写错看 `ConnectError`、把 read timeout 改成 `0.01` 看 `TimeoutException`，记录异常类名和 traceback 形态。
- [ ] 故意请求 `/status/404`，验证不写 `raise_for_status()` 时程序静默通过。
- [ ] 把 `/status/500` 交给你的重试函数，打印每次退避时长，确认是 0.5 / 1.x / 2.x。
- [ ] 提交。

**会踩的坑**

1. 每次请求都新建 `httpx.Client()`，连接不复用，慢且耗尽端口。一个进程一个 client，用 `with` 管理。
2. 只看 `resp.status_code` 不抛异常，然后把 HTML 错误页当 JSON 解析，报 `json.decoder.JSONDecodeError` 而你以为模型坏了。**先 raise_for_status，再解析 body**。
3. `resp.text[:200]` 进日志是好习惯，但把完整请求体（含用户隐私/内部数据）打进日志是安全事故。现在就定规矩：日志只记长度和摘要，不记内容。

**自测**

1. `httpx.Timeout(30.0)` 和 `httpx.Timeout(connect=5.0, read=30.0, ...)` 差别是什么？
2. 为什么 401 不该重试、429 该重试？
3. `raise_for_status()` 抛的是什么类型的异常，你的 `except` 该怎么写才不会把它误吞？

---

## 周二 · pydantic v2：把边界变成类型

**本质**：外部世界给你的是一坨 JSON 字典，内部代码想拿到的是有类型的对象。pydantic 在边界上做一次校验 + 转换，之后你的代码里不再有 `d["foo"]` 和拼错 key。

**Java 视角**：Jackson 的 `@JsonProperty` + `@Valid` + DTO 三件事，pydantic 用一种类声明一次做完，且**校验在解析时发生，不是事后调 validate()**。另一个对照：你的 Java DTO 里字段可空性靠 `Optional` 或 `@Nullable`，Python 里 `str | None` 就是类型的一部分。

**核心讲解**

*API 名字要认准 v2*：`model_validate(obj)`（不是 `parse_obj`，v1 写法）、`model_dump()` / `model_dump_json()`（不是 `dict()`）、`field_validator` + `@classmethod`（不是 `validator`）、配置写在 `model_config = ConfigDict(...)` 里（不是 `class Config`）。网上旧教程大量是 v1，照着抄必报错。

*`extra` 策略是设计决策*：`ignore` 容错、`forbid` 严格（模型多返回一个字段说明 schema 漂移，早点知道）、`allow` 几乎不该用。LLM 响应建议 `ignore`（服务商爱加字段），你自己定义的请求体用 `forbid`。

*校验失败给你的是地图，不是错误*：`ValidationError.errors()` 返回结构化列表（字段路径、类型、消息）。W05 你会把这个列表**原样回喂给模型让它自己修**——这是让结构化输出稳定的关键技巧，所以今天就把它打印出来看看长什么样。

**代码**

```python
import json
from datetime import datetime
from typing import Any, Literal

from pydantic import BaseModel, ConfigDict, Field, ValidationError, field_validator


class Usage(BaseModel):
    model_config = ConfigDict(extra="ignore")

    prompt_tokens: int = 0
    completion_tokens: int = 0

    @property
    def total_tokens(self) -> int:
        return self.prompt_tokens + self.completion_tokens


class ToolCall(BaseModel):
    model_config = ConfigDict(extra="ignore")

    id: str
    type: Literal["function"]  # 枚举值用 Literal，比 Enum 轻
    function: "FunctionCall"


class FunctionCall(BaseModel):
    model_config = ConfigDict(extra="ignore")

    name: str
    arguments: str  # 注意：这里是 JSON 字符串而不是 dict，需要二次解析

    @field_validator("arguments")
    @classmethod
    def _arguments_must_be_json_object(cls, v: str) -> str:
        parsed = json.loads(v or "{}")  # 解析失败直接抛 ValueError，pydantic 会包装成校验错误
        if not isinstance(parsed, dict):
            raise ValueError("arguments 必须是 JSON 对象")
        return v

    def parsed_args(self) -> dict[str, Any]:
        return json.loads(self.arguments or "{}")


class ChatMessage(BaseModel):
    model_config = ConfigDict(extra="ignore")

    role: Literal["system", "user", "assistant", "tool"]
    content: str | None = None
    tool_calls: list[ToolCall] | None = None

    def as_dict(self) -> dict[str, Any]:
        """送回 API 时用 exclude_none：多余字段会导致部分服务商 400。"""
        return self.model_dump(exclude_none=True)


class ChatResponse(BaseModel):
    model_config = ConfigDict(extra="ignore")

    id: str
    model: str
    choices: list[dict[str, Any]] = Field(min_length=1)
    usage: Usage | None = None


raw = json.dumps(
    {
        "id": "chatcmpl-1",
        "model": "some-model",
        "choices": [
            {
                "index": 0,
                "message": {
                    "role": "assistant",
                    "content": None,
                    "tool_calls": [
                        {
                            "id": "call_1",
                            "type": "function",
                            "function": {"name": "sql_query", "arguments": '{"sql": "select 1"}'},
                        }
                    ],
                },
            }
        ],
        "usage": {"prompt_tokens": 128, "completion_tokens": 12, "extra_field": "新字段"},
    }
)

resp = ChatResponse.model_validate_json(raw)  # 一步：解析 + 校验 + 构造对象
print(resp.usage.total_tokens)                 # 140：模型给了 total 而没给

msg = ChatMessage.model_validate(resp.choices[0]["message"])
call = msg.tool_calls[0]
print(call.function.name, call.function.parsed_args())  # sql_query {'sql': 'select 1'}

# 校验失败时看 errors()：这是你后面自动修复提示词的输入
bad = {"id": "x", "model": "m", "choices": [], "usage": {"prompt_tokens": "abc", "completion_tokens": 1}}
try:
    ChatResponse.model_validate(bad)
except ValidationError as exc:
    for e in exc.errors():
        print(e["loc"], e["type"], e["msg"])
```

**今天的动作**

- [ ] `uv add pydantic`，把上面代码跑通。
- [ ] 把周一拿到的 httpbin JSON 响应定义成一个模型并解析成功。
- [ ] 故意构造 3 种错误输入（缺字段 / 类型错 / 空数组），打印 `errors()` 输出，观察 `loc` 的嵌套路径格式。
- [ ] 给 `ChatMessage.as_dict()` 写一个断言：不含 `None` 字段。
- [ ] 提交。

**会踩的坑**

1. 把 `arguments` 当 dict 直接用。它是 **JSON 字符串**，必须先 `json.loads`，而且可能因为模型输出被截断而成不完整 JSON——W05 的自动修复就处理这个。
2. `model_dump()` 默认带出所有 None 字段。发给 API 时要用 `exclude_none=True`，否则部分服务商 400。
3. 在类里写 `class Config:`（v1 风格）——v2 里不报错但部分行为失效，最难查。一律 `model_config = ConfigDict(...)`。
4. `Optional[dict]` 校验失败时报错信息很长，不要慌，看 `loc` 定位字段。

**自测**

1. pydantic 的校验发生在什么时点？和 Java 的 `@Valid` + 手动 validator 比有什么不同？
2. `extra="forbid"` 在你自己的请求体上为什么是好事？
3. `Literal` 相比 `Enum` 的取舍是什么？

---

## 周三 · asyncio：单线程并发

**本质**：一个线程 + 一个事件循环，靠"在等 IO 时主动让出 CPU"实现并发。不是多线程，不是多核。

**Java 视角**：这是本周真正的门槛。对照关系是——`ExecutorService` 提交 10 个任务占 10 个线程；`asyncio` 提交 10 个协程只占 **1 个线程**，靠协程在 `await` 处挂起。好处是 1000 个并发不爆线程；代价是**任何一个同步阻塞调用都会卡死整个循环**，而且栈追踪变难读。CPU 密集任务在这里没有好处，要用 `ProcessPoolExecutor`（对应 Java 的 ForkJoinPool 场景）。

**核心讲解**

*"await 是唯一的让出点"（协作式调度）*：这是理解一切的钥匙。两个 `await` 之间没有抢占，所以你的代码不会被"中途换线程"，不需要锁；但也意味着一段 3 秒的 `time.sleep()` 会让整个进程停 3 秒。

*结构化并发*：`asyncio.gather` 并发跑并收集结果；`asyncio.TaskGroup`（3.11+）更严：任一子任务失败会自动取消其余，异常带 `ExceptionGroup`。**新代码优先 TaskGroup**，它替你处理了"忘 await"和"孤儿任务"这两类经典泄漏。

*并发上限*：没有 `Semaphore` 的 gather 等于同时打 100 个请求——你会拿到一堆 429。这和你 Java 里给线程池设最大线程数是同一件事。

*同步/异步不能混*：`async def` 里调用 `requests.get()` 或 `time.sleep()` 会静默串行化，你以为并发了其实没有。诊断方法就是本周五的动作之一。

**代码**

```python
import asyncio
import time

import httpx


async def fetch(client: httpx.AsyncClient, url: str, *, sem: asyncio.Semaphore) -> tuple[str, int]:
    async with sem:  # 并发闸门：同时最多 3 个请求在飞
        r = await client.get(url)
        return url, r.status_code


async def main() -> None:
    urls = [f"https://httpbin.org/delay/1?i={i}" for i in range(9)]

    # 版本一：同步串行（用 httpx 同步 client，思路一致）。9 × ~1s ≈ 9s
    start = time.perf_counter()
    with httpx.Client(timeout=30) as c:
        _ = [c.get(u) for u in urls]
    print(f"同步串行 {time.perf_counter() - start:.2f}s")

    # 版本二：异步并发 + 上限 3。ceil(9/3) × ~1s ≈ 3s
    start = time.perf_counter()
    sem = asyncio.Semaphore(3)
    timeout = httpx.Timeout(connect=5.0, read=30.0, write=10.0, pool=5.0)
    async with httpx.AsyncClient(timeout=timeout) as client:
        results = await asyncio.gather(*(fetch(client, u, sem=sem) for u in urls))
    print(f"异步并发(上限3) {time.perf_counter() - start:.2f}s，共 {len(results)} 条")

    # 版本三：TaskGroup —— 任一失败立刻取消其余，不留孤儿任务
    async with httpx.AsyncClient(timeout=timeout) as client:
        async with asyncio.TaskGroup() as tg:
            tasks = [tg.create_task(fetch(client, u, sem=sem)) for u in urls]
    print([t.result() for t in tasks][:2])

    # 逃生舱：只有同步 API 可用时，别阻塞事件循环，丢给线程
    def legacy_blocking_call() -> str:
        time.sleep(1)
        return "ok"

    print(await asyncio.to_thread(legacy_blocking_call))


asyncio.run(main())
```

**今天的动作**

- [ ] 跑上面的脚本，**记下三个数字**（同步 / 异步上限3 / 异步无上限），存进 `experiments/async-vs-sync.md`。
- [ ] 把 `Semaphore(3)` 改成 9，再改成 1，看总耗时变化，理解闸门与 429 的关系。
- [ ] 在 `async def fetch` 里塞一个 `time.sleep(0.5)`（注意：不是 `await asyncio.sleep`），重跑，观察耗时回到接近串行。**这个实验比任何解释都能让你记住阻塞是什么意思。**
- [ ] 用 TaskGroup 重写一遍，故意让第 3 个 URL 失败，看 `ExceptionGroup` 的结构与其余任务被取消的行为。
- [ ] 提交，并在实验笔记里写一句"什么时候该用异步、什么时候不该"。

**会踩的坑**

1. 在协程里调同步 client（`requests` / `httpx.Client`）——静默串行，性能损失且极难发现。表现是"我明明并发了，耗时却等于逐个相加"。
2. 忘写 `await`：`asyncio.gather(fetch(...))` 少了 await 会得到 coroutine 对象和一个 "coroutine was never awaited" 警告。Python 3.12 的警告很明确，别忽略。
3. `asyncio.run()` 只能在主线程调用一次；在 Jupyter 或已有循环里会报 "event loop is running"。学习阶段一律写成 `python -m` 跑的脚本，别在 notebook 里折腾。
4. 把 `asyncio.gather(*coros)` 当事务用——它没有回滚，任一失败其余照样跑完（要取消语义就用 TaskGroup）。

**自测**

1. 你的机器只有 1 个线程跑事件循环，为什么 9 个请求能并发？
2. `await asyncio.sleep(1)` 和 `time.sleep(1)` 在协程里的效果差别，具体到"其他任务此刻在干什么"。
3. TaskGroup 相比 gather 多保证了什么？

---

## 周四 · 配置与密钥

**本质**：密钥和配置不进代码、不进仓库、不进日志。听起来简单，但你会在 W05 反复复制提示词、W22 部署时把它彻底考验一遍。

**Java 视角**：等价于 Spring Boot 的 `application.yml` + 环境变量覆盖 + `@ConfigurationProperties` 绑定，只是 Python 没有内置这套，靠 `pydantic-settings` 实现同样的类型安全。

**核心讲解**

*优先级顺序*：显式传参 > 环境变量 > `.env` 文件 > 代码默认值。**代码默认值只放非敏感项**。`.env` 是给本地开发用的，生产环境只注入环境变量。

*`load_dotenv` 的时机*：必须在读 `os.environ` 之前。所以它放在程序入口或 settings 模块的顶部，不要散落在各处。

*密钥不进 git 之后，还要保证不进日志*：`SecretStr` 的 `repr` 是 `**********`，`model_dump()` 默认也会掩掉。用 `get_secret_value()` 才能取到明文——需要明文的地方就是你该小心的地方。

**代码**

```python
# .env.example —— 这个文件进 git，只有键名没有值
# LLM_API_KEY=
# LLM_BASE_URL=https://api.openai.com/v1
# LLM_MODEL=
# LOG_LEVEL=INFO
```

```python
# src/config.py
from functools import lru_cache

from pydantic import Field, SecretStr
from pydantic_settings import BaseSettings, SettingsConfigDict


class Settings(BaseSettings):
    model_config = SettingsConfigDict(
        env_file=".env",          # 本地自动读取
        env_file_encoding="utf-8",
        extra="ignore",           # 环境里多余变量忽略，缺必填直接报错
    )

    llm_api_key: SecretStr | None = None
    llm_base_url: str = "https://api.openai.com/v1"
    llm_model: str = "gpt-4o-mini"
    max_agent_steps: int = Field(default=8, ge=1, le=32)  # 范围校验：防止跑飞
    request_timeout: float = Field(default=60.0, gt=0)
    log_level: str = "INFO"

    @property
    def has_llm_key(self) -> bool:
        return bool(self.llm_api_key)


@lru_cache
def get_settings() -> Settings:
    """单例：重复实例化会重复读文件，也让测试难以覆盖。"""
    return Settings()  # type: ignore[call-arg]


if __name__ == "__main__":
    s = get_settings()
    print(s.llm_model, s.max_agent_steps)
    print(s.llm_api_key)  # SecretStr('**********') —— 故意打印也泄不了
    if not s.has_llm_key:
        raise SystemExit("缺少 LLM_API_KEY，先写进 .env")
```

```gitignore
# 追加到 .gitignore
.env
.env.*
!.env.example
.venv/
__pycache__/
*.pyc
.pytest_cache/
.ruff_cache/
data/
*.sqlite
```

```bash
uv add pydantic-settings
uv run python -m src.config
```

**今天的动作**

- [ ] `.env.example` 与 `src/config.py` 建好，`.gitignore` 补全。
- [ ] 建一个 `.env`（暂时随便填），跑 `python -m src.config` 验证能读到。
- [ ] 验证优先级：环境变量 `LOG_LEVEL=DEBUG uv run python -m src.config` 覆盖 `.env`。
- [ ] 把 `max_agent_steps` 改成 999，看 pydantic 的 `le=32` 报错——**这个范围校验后面防止你一夜烧掉额度**。
- [ ] 用 `git log --stat` 确认 `.env` 从未被提交过（现在还没提交，正好养成检查习惯）。
- [ ] 提交。

**会踩的坑**

1. `.env` 已提交后才加 `.gitignore`——`gitignore` 不管已跟踪文件，要 `git rm --cached .env`。**如果 token 进过 commit，立刻去服务商后台吊销重签**，删提交历史不算撤回泄漏。
2. 忘了 `extra="ignore"`，环境里无关变量导致启动报错。
3. 在模块顶层 `settings = Settings()` 然后被到处 import——测试时无法覆盖。用 `get_settings()` 函数 + 参数注入。

**自测**

1. 为什么 `.env` 提交过一次就必须吊销 token，而不能只 `git rm`？
2. `SecretStr` 在什么情况下仍会泄漏明文？
3. 生产环境为什么只该用环境变量而不是 `.env`？

---

## 周五 · pytest

**本质**：测试是 Python 项目里唯一可靠的"编译器"。没有 javac 替你查类型和调用契约，测试就是你的构建门。

**Java 视角**：JUnit 5 的 `@Test` / `@ParameterizedTest` / `@BeforeEach` 分别对应：以 `test_` 开头的函数、`@pytest.mark.parametrize`、fixture。差别是**没有继承、没有注解魔法，依赖注入靠函数参数名匹配**——参数名写个 fixture 名，pytest 就注进来，靠约定而非声明。

**核心讲解**

*fixture 是可以带参数的资源工厂*：`@pytest.fixture` 的函数 `yield` 前是 setup、后是 teardown。`scope="module"` 让一个模块只建一次（比如向量库连接）。这个概念后面 W14 的评测集会直接复用。

*参数化替代数据驱动测试*：Java 的 `@CsvSource` 在这里是 `parametrize("name,expected", [...])`，比写循环更清晰且失败能定位到具体一条。

*测试不要碰真实网络和真实模型*：单元测试里调 LLM API 会让你每次跑测试都花钱、且结果不稳定。用 fixture 返回假响应对象，或在 W21 的评测集才允许真实调用（那是集成测试，标 `@pytest.mark.slow`）。

**代码**

```python
# tests/conftest.py —— 目录级共享 fixture，不需要 import
import pytest


@pytest.fixture
def fake_llm_response() -> dict:
    return {
        "id": "chatcmpl-test",
        "model": "test-model",
        "choices": [{"index": 0, "message": {"role": "assistant", "content": "pong"}}],
        "usage": {"prompt_tokens": 5, "completion_tokens": 3},
    }


@pytest.fixture(scope="module")
def sample_docs() -> list[str]:
    return ["订单超时 30 分钟自动取消", "退款需人工审批", "订单超时定义"]
```

```python
# test_w02.py
import pytest

from src.pyforjava.strings import chunked  # 上周写的工具，现在有了测试保护


@pytest.mark.parametrize(
    "size,expected",
    [(1, 5), (2, 3), (5, 1), (10, 1)],
)
def test_chunked_sizes(size: int, expected: int) -> None:
    assert len(chunked(list(range(5)), size)) == expected


def test_chunked_rejects_zero() -> None:
    with pytest.raises(ValueError, match="size"):
        chunked([1, 2], 0)


def test_pydantic_roundtrip(fake_llm_response: dict) -> None:
    from src.llm.models import ChatResponse  # W02 周二建的模型

    resp = ChatResponse.model_validate(fake_llm_response)
    assert resp.usage and resp.usage.total_tokens == 8
    assert resp.choices[0]["message"]["content"] == "pong"


@pytest.mark.slow
def test_real_api_is_not_called_in_unit_tests() -> None:
    pytest.skip("真实调用留给 W21 评测集，单元测试不花钱")
```

```bash
uv add --dev pytest pytest-asyncio
uv run pytest -q
uv run pytest -k "chunked" -v      # 按名字筛
uv run pytest --strict-markers     # 未注册的 mark 直接报错，避免拼错静默失效
```

`pyproject.toml` 补上：

```toml
[tool.pytest.ini_options]
testpaths = ["tests"]
markers = ["slow: 真实外部调用或耗时用例，默认不跑"]
addopts = "-m 'not slow' --strict-markers"
```

**今天的动作**

- [ ] 建 `tests/conftest.py` 与一个测试文件，`uv run pytest` 全绿。
- [ ] 给 W01 的 `group_by`、`chunked` 各补到 3 个测试（含一个异常路径）。
- [ ] 用 `parametrize` 写一个"给定非法输入 → 期望 ValidationError"的表驱动测试。
- [ ] 给 `async def fetch` 写一个异步测试（`pytest-asyncio`，标 `@pytest.mark.asyncio`，配 `asyncio_mode = "auto"`）。
- [ ] **故意把一个断言写错**，读懂 pytest 的 diff 输出格式，学会看 `E   assert` 那一段。
- [ ] 提交。

**会踩的坑**

1. 测试文件叫 `test_*.py` 之外的名字，pytest 完全不收。旧代码里 `xxx_test.py` 要显式配 `python_files`。
2. `addopts` 里写了 `-m 'not slow'`，结果你新加的 `@pytest.mark.slow` 一个都不跑，误以为测试很快——`--strict-markers` + `-m slow` 显式跑一次确认。
3. 在测试里 `import httpx` 然后真发请求，CI 上不稳定的第一个来源。用 fixture 把 client 换成假的。

**自测**

1. fixture 的 `yield` 前后分别何时执行？`scope="module"` 什么时候重建？
2. pytest 靠什么决定参数注进来的是哪个 fixture？
3. 为什么单元测试严禁调用真实 LLM API？例外场景在哪一周出现？

---

## 周六 · 第一个真实模型调用 + 主仓库定型（120 分钟）

分两段，中间离开屏幕 10 分钟。

**第一段 55' · 拿到 key 并跑通一次调用**

1. 选一家服务商（任一即可，后面所有讲义都写成不依赖特定 SDK）：需要有稳定的 chat completions、tool calling、structured output 能力，以及**可查的用量账单页面**。
2. 建 key → 写进 `.env` 的 `LLM_API_KEY` → **不要提交**。
3. 用 `httpx` 直接发一次原始请求（刻意不用官方 SDK，让你看得见线上到底传了什么）：

```python
# scripts/first_call.py
import httpx
from pydantic import BaseModel

from src.config import get_settings

s = get_settings()
if not s.has_llm_key:
    raise SystemExit("先在 .env 写 LLM_API_KEY")

payload = {
    "model": s.llm_model,
    "messages": [
        {"role": "system", "content": "你是一个严格的助手，只输出 JSON。"},
        {"role": "user", "content": "把这句话抽成 JSON：张三 32 岁，在北京做 Java 开发。"},
    ],
    "temperature": 0,
    "max_tokens": 200,
    # 字段名（如 response_format / reasoning_effort）随服务商与模型不同，见其文档
}

with httpx.Client(
    base_url=s.llm_base_url,
    headers={"Authorization": f"Bearer {s.llm_api_key.get_secret_value()}"},
    timeout=httpx.Timeout(connect=5.0, read=s.request_timeout, write=10.0, pool=5.0),
) as client:
    resp = client.post("/chat/completions", json=payload)
    resp.raise_for_status()

data = resp.json()
print("原始响应键：", list(data.keys()))
print("内容：", data["choices"][0]["message"]["content"])
print("用量：", data.get("usage"))  # 若为 None，说明服务商把 usage 放在别处，去文档确认
```

4. 把原始响应完整粘进 `experiments/first-call.md`，并**逐字段标注**：哪个是消息、哪个是用量、哪个是结束原因（`finish_reason` 为 `length` 表示被 max_tokens 截断——记住这个，W03 会用到）。
5. 去账单页确认这次实际花了多少钱，记下来。后面 W21 的 30 条评测集要跑几十次，你现在需要知道自己单次成本的数量级。

**第二段 55' · 主仓库定型**

- [ ] 目录按 [07-每日执行模板.md](../plan/07-每日执行模板.md) 最后一节的结构建全（`src/llm`、`src/prompts`、`src/tools`、`src/agent`、`src/retrieval`、`evals`、`experiments`、`projects`）。
- [ ] 把本周代码归位：`models.py` 进 `src/llm/`，`config.py` 留在 `src/`，W01 工具进 `src/pyforjava/`。
- [ ] README 写清：这是什么、环境怎么起（uv 三条命令）、`.env.example` 说明、目录结构、当前进度。
- [ ] `uv run pytest` 全绿 + `uv run pyright` 零报错，作为后续所有提交的门槛。
- [ ] 本周状态与三行记录写进 `learning-log.md`。

**会踩的坑**

1. 报 401/403：key 复制或 base_url 区域不对（同一服务商不同接入点的 base_url 不同）。先把 key 打印长度（不是内容）确认没截断。
2. 报 404 且提示 model not found：模型名写错了，去控制台或"模型列表"接口拿准确字符串，别猜。
3. 新账号没有额度就报 402/权限错误——先充值或领额度再折腾代码。
4. 第一次调用就把 `temperature` 拉高玩——本周要的是**确定性**，方便你对比响应结构。

**自测**

1. `finish_reason` 有哪几种取值，各自说明什么？
2. 为什么输入 token 和输出 token 单价不同？这次调用总成本怎么算？
3. 如果响应里 `usage` 是 None，你第一步查什么？

---

## 本周术语卡

| 术语 | 一句话解释 | 为什么重要 |
|---|---|---|
| `httpx.Client` / `AsyncClient` | 同步与异步两套 HTTP 客户端，内含连接池 | 一个进程一个实例；两者不可混用 |
| read timeout | 等待响应字节的最大间隔 | LLM 场景要放大，流式按 chunk 间隔计 |
| 指数退避 + 抖动 | 每次重试间隔翻倍再加随机值 | 避免多进程同步撞限流 |
| pydantic 校验 | 解析时一次性做类型转换与约束检查 | 边界一次校验，内部代码不再写防御式判断 |
| `ValidationError.errors()` | 结构化的校验失败列表（含字段路径） | 直接可作为"让模型自己修"的输入 |
| `SecretStr` | repr 自动掩码的密钥类型 | 打印/转储时不泄明文 |
| 事件循环 | 单线程调度协程的中心 | `await` 是唯一让出点 |
| `Semaphore` | 并发闸门 | 没上限的并发 = 429 |
| `TaskGroup` | 3.11+ 结构化并发，失败即取消全部 | 消除孤儿任务与忘 await |
| `asyncio.to_thread` | 把同步阻塞调用丢进线程 | 唯一正当的"逃生舱" |
| fixture | 按参数名注入的测试资源工厂 | 替代 `@BeforeEach`，可带作用域与 teardown |
| `parametrize` | 表驱动测试 | 失败能定位到具体一行 |
| `finish_reason` | 模型结束原因 | `length` = 被截断，不是答完了 |

## 延伸阅读（通勤时看，不占 1 小时）

- pydantic v2 迁移指南的 "Changes" 一节（专门用来纠正 v1 肌肉记忆）。
- asyncio 官方文档 "Coroutines and tasks" 一节，读完能解释 `TaskGroup` 为何更安全。
