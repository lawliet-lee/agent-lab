# W18 · MCP（Model Context Protocol）

> 本周在计划中的位置：给项目二装上"协议层"。你要写出自己的第一个 MCP server，把内部能力（只读 SQL / 内部 API / 部署单查询）暴露成任何宿主都能调用的标准工具——这是你简历上"有平台意识"最硬的证据。

## 本周导航

| 日 | 主题 | 今天的产出物 |
|---|---|---|
| 周一 | 规范概览：M×N 集成爆炸 + tools/resources/prompts 三类原语 | `experiments/w18-primitive-mapping.md`：项目二能力逐项归类 |
| 周二 | 写 MCP 客户端 | `src/mcp/client_probe.py`：连一个现成 server，列工具、调用、拿结果 |
| 周三 | 写 MCP server | `mcp_servers/ops_readonly.py`：一个真实能力的工具 + 描述好坏对照 |
| 周四 | 治理：鉴权 / 超时 / 错误约定 / 可见性 / 确认门 | 同一 server 加上治理层，附五条硬性规则清单 |
| 周五 | 项目二工具层换成 MCP 调用 | 直连函数 vs MCP 调用的对比记录（延迟、可调试性、耦合） |
| 周六 | 端到端联调 + 局限笔记 | 演示跑通 + `experiments/w18-what-mcp-does-not-solve.md` |
| 周日 | 休息，不许补课 | 无 |

**开工前 20 分钟核对版本**（任务表要求）。以下是已核实的当前事实，本周会反复用到：

- 规范当前修订版 **2026-07-28**；此前 2025-11-25、2025-06-18、2025-03-26。SSE 传输在 2025-03-26 就被 **Streamable HTTP** 取代，别在新项目上用。2026-07-28 这一版移除了连接握手/会话/`ping`，废弃了 roots、服务器发起的 sampling 与协议内 logging，进度通知改为仅 server→client。
- Python SDK 已是 **v2**（写这周时最新 2.3.0）：`FastMCP` 改名 **`MCPServer`**（`from mcp.server import MCPServer`），旧的 `mcp.server.fastmcp` 导入路径**已删除**而不是废弃，`ImportError` 就是它。装包 `uv add "mcp[cli]"`，SDK 要求 Python **3.10+**（你 W01 装的 3.12 直接满足，别为了这门课去动版本）。
- 你在 W2 学的 `httpx` 在 SDK v2 的传输层里换成了 `httpx2`（API 同形）。你自己项目里的异步 HTTP 仍用 `httpx.AsyncClient`，两者别混。

## 周一 · 它解决的不是"调用工具"，是集成爆炸

**本质**：M 个模型宿主 × N 个企业内部系统，点对点私有对接是 M×N 次；引入一层标准协议后变成 N 次实现 + M 次接入。**MCP 解决的是集成成本，不是智能**。它不保证模型选对工具、不保证快、不替你鉴权。

**Java 视角**：这就是 SPI / JDBC / OpenAPI 的位置——先定义"能力怎么被描述和调用"，实现方与消费方各自独立演化。你也可以理解成"给 LLM 用的 gRPC：IDL 是 JSON Schema，传输是 JSON-RPC 2.0"。

**核心讲解**

协议是一次 JSON-RPC 2.0 会话，起手做**能力协商（capability negotiation）**：双方声明自己支持哪些原语，没声明的一律不能调。客户端侧对象是 host（Claude Desktop、Cursor、你自己的 Agent）→ client（一个连接会话）→ server（能力提供方）。

三类原语**不能混为一谈**，差别在"谁决定用它"：

| 原语 | 谁触发 | 语义 | 关键差异 |
|---|---|---|---|
| tools | **模型**决定调用 | 有副作用或取数的动作 | 输入是 JSON Schema（由类型注解生成），失败可以是模型可读的 error 结果 |
| resources | **应用**决定加载 | 只读上下文/数据 | 按 **URI** 寻址而非函数名；`resources/list` 不调用你的函数，只有 `resources/read` 才调用 |
| prompts | **用户**从菜单里选 | 消息模板 | 参数是"扁平的命名字符串列表"，没有 JSON Schema；缺必填参数直接抛 JSON-RPC 错误 `-32602`，没有"给模型看的错误结果"这一说 |

三条常被误解的边界（记住能省你几小时）：① **resource 不是"只读的 tool"**——区别是控制权：把该由人/应用决定要不要注入的上下文做成 tool，模型会自作主张地去读；反过来把有副作用的动作做成 resource，宿主可能把它当静态数据缓存。② **模板化资源**（`@mcp.resource("users://{user_id}/profile")`）离开 `resources/list`，出现在 `resources/templates/list`，它是"模式"不是"地址"。URI 里的占位符名必须与函数参数名完全一致，否则**导入时就报 `ValueError`**——这种"启动即失败"的设计是好事，别绕。③ **prompt 不参与模型决策**。它是给人点的按钮，别指望用它做系统提示词的集中管理（那是你 `src/prompts/` 的职责）。

**代码**

```python
"""周一的归类练习：项目二每个能力显式归类，写不出 owner 就说明你还没想清楚。"""
from __future__ import annotations
from enum import Enum
from pydantic import BaseModel, field_validator
class Owner(str, Enum):
    MODEL = "model"      # tools：模型决定调用
    APPLICATION = "app"  # resources：应用决定加载
    USER = "user"        # prompts：用户从菜单选
class Capability(BaseModel):
    name: str
    owner: Owner
    side_effect: bool = False
    uri_or_schema_hint: str = ""
    @field_validator("name")
    @classmethod
    def snake_case(cls, v: str) -> str:
        if not v.replace("_", "").islower():  # 命名风格不一致会拉低模型选工具的正确率
            raise ValueError("工具/资源名统一 snake_case")
        return v
PROJECT_TWO = [
    Capability(name="sql_readonly_query", owner=Owner.MODEL, uri_or_schema_hint="只读、带 limit 与超时"),
    Capability(name="deploy_ticket_status", owner=Owner.MODEL),
    Capability(name="incident_runbook", owner=Owner.APPLICATION, uri_or_schema_hint="runbook://service/{service}"),
    Capability(name="postmortem_template", owner=Owner.USER),
]
# 反例：Capability(name="restart_pod", owner=Owner.APPLICATION) —— 有副作用的东西绝不能是 resource
```

**今天的动作**

- [ ] 读官方规范概览 + architecture 页（通勤时段读完，晚上只做归类）。标记你不认识的术语，别做笔记抄写。
- [ ] 把项目二全部能力列成 `Capability` 清单，逐项定 owner；owner 判据写在注释里而不是靠感觉。
- [ ] 挑 2 个你原本想做成 tool 的"只读上下文"改判成 resource，写清改判理由。
- [ ] 在 `experiments/w18-primitive-mapping.md` 里回答一句话：本项目里哪个能力**不该**做成 MCP（答案通常是"延迟敏感且只有本地用到的内部函数"）。git commit。

**会踩的坑**

1. **把三类原语当"都能当工具"**。表现：所有东西都注册成 tool，宿主 UI 里资源面板是空的，模型上下文塞满它不该主动调的只读数据。诊断：数一下 `tools/list` 返回条目与你的能力清单是否一一对应。
2. **拿 prompt 管系统提示词**。表现：prompt 在客户端菜单里躺着没人点。它面向人，不面向模型。
3. **误以为 resource 列表会调用你的函数**。它只在 read 时调用；所以列表要轻、别在 `resources/list` 这条路径上做重活。
4. **照抄网上 v1 教程**。`FastMCP`、`mcp.server.fastmcp`、`ctx.info()`、`ctx.session.create_message()` 这些写法在 2026 版里要么改名要么被废弃，第一次运行就报 `ImportError` 或 `MCPDeprecationWarning`。

**自测**：① 一句话解释"M×N 集成爆炸"，并说出 MCP 把哪个乘号消掉了、哪个没消掉。② tools/resources/prompts 各自的触发者是谁？举一个你会误用成 tool 的 resource。③ 模板资源与具体资源在客户端看到的列表有什么不同？

## 周二 · 写客户端：连一个现成 server

**本质**：客户端代码是"能力发现 + 调用 + 结果解读"三件事。最容易被忽略的是第三件：结果有两个通道（给模型的 `content` 与给程序的 `structured_content`），外加一个必须先看的风向标 `is_error`。

**Java 视角**：像先用 OpenAPI 文档做联调——`list_tools()` 就是拉 schema，`call_tool()` 就是发请求。`is_error` 对应"HTTP 200 但响应体里说业务失败"，你写过电商接口就认得这个坑。

**核心讲解**

`Client` 一个对象、一个生命周期：构造只挑传输，**`async with` 才真正连接与协商**；退出即断开，块外访问会抛 `RuntimeError: Client must be used within an async context manager`。`Client` 的入参按类型解析：URL 字符串 → Streamable HTTP；`StdioServerParameters` → 拉起子进程走 stdio；server 对象本身 → 进程内直连（测试用，**跳过 HTTP 层也跳过鉴权**）。进块后四个属性直接可读：`server_info`、`server_capabilities`（没声明的能力是 `None`）、`protocol_version`（这里应该是 `"2026-07-28"`）、`instructions`。

结果解读三条硬规则：**先看 `is_error` 再用 `structured_content`**；`content` 是内容块 union（`TextContent`/`ImageContent`/`ResourceLink`/`EmbeddedResource`），必须先 `isinstance` 收窄再取 `.text`；调一个不存在的工具**不抛异常**，而是返回 `is_error=True` 且 `content` 里写着 `Unknown tool: xxx`——只有 server 回了 JSON-RPC error 才会在客户端抛 `MCPError`。分页：每个 `list_*` 都有 `cursor=`，结果带 `next_cursor`，为 `None` 才代表拿全。还有一条会咬人的 stdio 规则：**子进程不继承你的环境变量**，只给一份白名单（POSIX 下 `HOME/LOGNAME/PATH/SHELL/TERM/USER`），server 要的 API key 必须用 `env=` 显式传。

**代码**

```python
from __future__ import annotations
import asyncio
from mcp import Client, StdioServerParameters
from mcp.types import TextContent
# 官方参考 server（文件系统类）。2026 年包名/启动命令请核对官方 servers 仓库，
# 但下面这套调用形态是 SDK v2 的稳定写法。
SERVER = StdioServerParameters(
    command="npx",
    args=["-y", "@modelcontextprotocol/server-filesystem", "./docs"],  # 换成你的仓库绝对路径
    env={},  # 需要密钥时在这里显式传：子进程不会继承你的 shell 环境
)

async def probe() -> None:
    async with Client(SERVER) as client:
        print(client.server_info, client.protocol_version)  # 版本不对，后面的怪事都有了解释
        caps = client.server_capabilities
        assert caps.tools is not None, "这个 server 没声明 tools 能力"
        for t in (await client.list_tools()).tools:
            # input_schema 就是模型看到的契约：缺描述的工具在这里就能被发现
            print(f"{t.name}  required={t.input_schema.get('required')}")
        result = await client.call_tool("read_file", {"path": "plan/W16.md"})
        if result.is_error:  # 硬规则：先判错，再取数据
            print("工具失败：", [b.text for b in result.content if isinstance(b, TextContent)])
            return
        text = "\n".join(b.text for b in result.content if isinstance(b, TextContent))
        print(f"读到 {len(text)} 字符；结构化通道={result.structured_content}")

if __name__ == "__main__":
    asyncio.run(probe())
```

**今天的动作**

- [ ] 装好 `mcp[cli]`，先跑 `uv run mcp dev server.py`（Inspector，需要 `npx`）确认环境可用，再跑上面的 `probe()`；打印 `server_capabilities` 与 `protocol_version`，核对协商到了哪个版本。
- [ ] 故意调一个不存在的工具，观察它返回 `is_error=True` 而不是抛异常；再写一段"先判 `is_error`"的统一处理。
- [ ] 用 `read_resource` 读一个具体 URI，对比 `list_resources()` 与 `list_resource_templates()` 两个列表的差异。
- [ ] 把三个工具的 `description` 原文抄进笔记，晚上判断哪条写得差。git commit。

**会踩的坑**

1. **`Client("http://...")` 后直接调方法**。构造不连接，`async with` 才连；忘记进块就 `RuntimeError`。
2. **拿 `result.content[0].text` 当万能**。第一个块可能是 `ImageContent`（没有 `.text`），类型检查器都会拦你，运行期 `AttributeError` 就是这么来的。
3. **业务失败被当成功**。server 里 `raise` 出来的 `ToolError` 会变成 `is_error=True` 的普通结果，如果你没看这个标志，等于把错误数据喂进上下文。
4. **`headers=`/`timeout=` 传给 `streamable_http_client`**。v2 的签名是 `url / http_client / terminate_on_close / max_sse_event_size`，headers 与超时要挂在你自己的 `httpx2.AsyncClient` 上；传错参数直接 `TypeError`。

**自测**：① `Client` 构造与 `async with` 各自的职责是什么？② server 端 `raise` 出来的工具错误，客户端会抛异常吗？该看哪个字段？③ 为什么 stdio server 里 `os.environ["MY_KEY"]` 常常是空的？

## 周三 · 写 server：把内部能力暴露成标准工具

**本质**：server 侧你写的每个工具，都是给模型看的一份接口文档 + 一段真实代码。**工具描述的质量直接决定模型用不用得对**——这不是文风问题，是功能问题。

**Java 视角**：类型注解生成 JSON Schema 这件事，等于 FastAPI 里"函数签名就是 OpenAPI 文档"。SDK 文档也明说：`a: int, b: int` **就是** schema，你不写一行 JSON。

**核心讲解**

`MCPServer("名字")` + `@mcp.tool()` 装饰普通函数：名字取函数名、描述取 docstring、参数取类型注解。默认值让参数变可选；`Annotated[..., Field(description=..., ge=, le=)]` 加描述与约束；`Literal[...]` 变枚举；参数直接用一个 Pydantic 模型可以接结构化 body（你的函数收到已校验的实例）。

约束不是装饰：客户端传 `limit=999` 而你写了 `le=50`，SDK 在**你的函数运行之前**就拒绝，并把 "Input should be less than or equal to 50" 作为工具错误回给模型——模型读到它会自己改参数重试。**你写一次 `le=50` 就买到一个自我纠错的 Agent**，这也意味着"别在自己的工具里重复校验类型注解已经管的事"。返回值声明成模型就生成 `output_schema`，客户端能拿到 `structured_content`。

描述怎么写。坏描述的特征是"只说是什么，不说何时用、何时别用、返回什么、边界在哪"：坏 = `"""查询数据库。"""` → 模型不知道能查什么表、能不能写、结果多大，于是要么不敢用要么乱用；好 = `"""只读查询生产元数据库，返回指定服务的近 N 小时错误率时序。仅用于已定位到具体服务后的错误率确认；跨服务聚合或写操作请改用 ticket 工具。单次最多 5000 行，查询超 15 秒会被上游中断。"""`

好描述四要素：**做什么 / 什么时候该用（以及什么时候不该用）/ 输入输出的形状与量级 / 失败与限制**。`Field(description=...)` 补每个参数语义（模型会读），枚举用 `Literal` 而不是在描述里写"可选值有……"。

注解（`ToolAnnotations`）是给宿主的行为提示：`read_only_hint`、`destructive_hint`、`idempotent_hint`、`open_world_hint`。**硬性规则：规范明确要求客户端把这些注解当不可信输入，它们是提示不是安全**。"我是只读的"必须靠数据库账号权限与 `enforce()` 类校验实现，`read_only_hint=True` 只用来让宿主少弹一次确认框。

**代码**

```python
from __future__ import annotations
from typing import Annotated, Literal
from pydantic import BaseModel, Field
from mcp.server import MCPServer
from mcp.server.mcpserver.exceptions import ToolError
from mcp.types import ToolAnnotations
mcp = MCPServer("ops-readonly")  # 构造函数只描述"这是什么"：name/version/instructions

class ErrorRatePoint(BaseModel):
    ts: str
    error_rate: float
    sample_size: int

@mcp.tool(
    title="Query service error rate",
    annotations=ToolAnnotations(read_only_hint=True, open_world_hint=False, idempotent_hint=True),
)
async def query_error_rate(
    service: Annotated[str, Field(description="服务名，必须与注册中心一致，例如 order-service。")],
    hours: Annotated[int, Field(ge=1, le=72, description="回看小时数。超过 72 请改用离线表。")] = 6,
    bucket: Literal["minute", "hour"] = "hour",
) -> list[ErrorRatePoint]:
    """只读查询生产元数据库，返回指定服务的错误率时序。
    用途：仅在已经定位到具体服务后用来确认错误率量级。
    不要用于：跨服务聚合（会超时）、任何写操作（本工具没有写权限）。
    输出：每点含 ts / error_rate / sample_size；单次最多 5000 行，采样数 <100 时结果不可信。
    失败：上游查询超 15 秒会中断，返回可被你修正的错误信息。
    """
    rows = await _query(service, hours, bucket)  # 真实取数：async def 工具由 SDK await，普通 def 跑在线程里
    if not rows:
        # ToolError 的消息模型读得到：必须写"怎么改对"，不是只写"失败了"
        raise ToolError(f"service={service!r} 不在注册中心，先用 list_services 拿到准确服务名。")
    return [ErrorRatePoint(**r) for r in rows]

if __name__ == "__main__":
    mcp.run(transport="stdio")  # 部署用 transport="streamable-http"；传输选项只写在 run() 里
```

**今天的动作**

- [ ] 选**一件你真有价值的能力**（只读 SQL / 内部 API / 部署单查询三选一），别贪多，先做 1 个工具。
- [ ] 用 `@mcp.tool()` + `Annotated/Field/Literal` 定义它，返回类型声明成 Pydantic 模型，让 `output_schema` 自动生成。
- [ ] 写坏描述与好描述各一版，用同一个真实问题在 Inspector 里跑，记录模型选对工具/填对参数的次数差异。
- [ ] 用 `ToolError` 覆盖三种失败：参数可修正的（提示怎么改）、上游超时的、无权限的。确认每条消息都写了"下一步怎么做"。
- [ ] `uv run mcp dev mcp_servers/xxx.py` 手工调 5 次，把 5 次结果贴进笔记。git commit。

**会踩的坑**

1. **描述写成人类文档**。表现：模型反复用错工具或漏填参数。诊断：在 Inspector 里只喂问题不喂提示，看它是否选对工具、参数是否越界。
2. **`mcp.run(port=8000)` 或 `MCPServer(port=...)`**。传输选项（host/port/streamable_http_path/json_response/stateless_http/max_request_body_size/session_idle_timeout/max_sessions）只在 `run()` 上，`log_level`/`debug` 在构造函数上。写反了 Python 立刻 `TypeError`。
3. **在工具里重复校验类型注解**。SDK 已在调用前按 schema 拒绝；你重复校验只会让错误消息变模糊。
4. **`print()` 打进 stdio**。stdout 就是协议线，会污染数据流。日志一律走 `logging`（它 flush 到 stderr），协议内 `ctx.log()` 那套在 2026-07-28 版已废弃。

**自测**：① 为什么 `le=50` 这类约束能让 Agent 自我纠错？错误回给谁、在哪个通道？② 好描述的四要素是什么？给你今天的工具逐条对一遍。③ `read_only_hint=True` 与安全有什么关系（正确答案：几乎没有，说明为什么）？

## 周四 · 治理：鉴权、超时、错误分层、可见性、确认门

**本质**：MCP server 一旦上线就是网络服务，治理项跟 Java 后端一模一样：认证、授权、超时、错误契约、危险操作审批。**协议本身不给你安全，它只给你标准。**

**Java 视角**：Streamable HTTP 上的 MCP server 就是一个 OAuth 2.1 资源服务器——它不登录、不发 token，只做一件事：看每个请求的 `Authorization` 头判断令牌是否有效。这跟你现在写的资源服务完全同构。

**核心讲解**

五条硬性规则，不是建议：

1. **鉴权**。实现 `TokenVerifier`（一个异步方法 `verify_token(token) -> AccessToken | None`），与 `AuthSettings(issuer_url, resource_server_url, required_scopes, validate_token_resource)` **成对**传给 `MCPServer`——只传一个会直接 `ValueError`。SDK 会自动发布 RFC 9728 的 `/.well-known/oauth-protected-resource/mcp` 元数据，并用带 `resource_metadata` 指针的 401 拒绝未授权请求。**`token.scopes` 就是你做工具可见性的依据；`get_access_token()` 在任意 handler 里拿到调用者身份（`client_id / scopes / subject / expires_at / claims`）**。
   ⚠️ 关键边界：**鉴权只存在于 HTTP 传输。stdio 没有 `Authorization` 头，管道永远不带 token；进程内 `Client(mcp)` 也绕过 HTTP 层——它们的信任边界是"谁启动了进程"**。别以为写了 `token_verifier` 就全局安全了。
2. **超时**。SDK 自带的 `httpx2.AsyncClient` 默认 30 秒（连接/写/池）与 300 秒（读，因为可能长持流）。你自建 client 时必须显式保留 `timeout=`，否则落到 httpx 的 5 秒默认，一次稍长的工具调用就变成读超时。**协议没有 per-call timeout 字段**，客户端侧的每次调用超时要自己包 `asyncio.timeout`。**server 侧还要有自己的上游超时**：`max_request_body_size`（默认 4 MiB）、`session_idle_timeout`、`max_sessions` 都有默认值，压测时按需调。
3. **错误返回约定：协议层错误与内容层失败必须分开**。三选一按"模型能否避免这次失败"判定：能避免 → `raise ToolError(msg)`（`mcp.server.mcpserver.exceptions`），结果 `is_error=True`、消息进 `content` 给模型看；不能避免（缺能力、server 状态不对、调用方跳了必经步骤、越权）→ `raise MCPError`（`from mcp`），整个请求变成 JSON-RPC 错误，模型什么都看不到，只有宿主看到；其他异常 → 视为崩溃，模型只看到 `Error executing tool <name>`，**traceback 留在服务端日志**（这是刻意的：不要把内部结构泄露给模型）。资源没有"半个结果"这一说：只有协议路径，缺 URI 用 `ResourceNotFoundError`。**永远不要 `return` 一串错误文本。**
4. **工具可见性按角色控制**。做法是"两屏"：**列表层**按角色动态注册/摘除工具（`mcp.add_tool(...)` / 运行时移除，再通知宿主重新 `tools/list`），**执行层**必须再查一次 scope 并拒绝。硬性规则：**隐藏工具不是安全控制，只是降噪**。真正的门是 handler 里的 scope 检查——`tools/list` 会被缓存、模型会幻觉调用、别的客户端能直连，列表拦不住。
5. **危险操作确认门**。有副作用的动作要么声明 `destructive_hint` 并在**代码里**要求确认，要么用协议层的 elicitation 问用户。SDK v2 里稳的做法是 **resolver**：参数写 `Annotated[ElicitationResult[T], Resolve(fn)]`，resolver 返回 `Confirm(...)`（不需要问，零往返）或 `Elicit(问题, 模型)`（要问才弹），工具体里 `match` 四种结局——accept 且答案为真 / accept 但答案为假 / decline / cancel 含义各不相同，别合并成"失败了"。注意 `ctx.elicit()` 是 server→client 反向请求，2026-07-28 的连接上没有反向通道，需要客户端 `mode="legacy"`；所以优先用 resolver，它两种时代都能跑。另：`json_response=True` 时没有反向通道，会抛 `NoBackChannelError`。
6. **别把密钥放进 resource URI 或工具参数**：它们会原样进模型上下文和宿主日志。要传的是资源 ID，不是凭据。

**代码**

```python
from __future__ import annotations
from typing import Annotated
from pydantic import AnyHttpUrl, BaseModel
from mcp import MCPError
from mcp.server import MCPServer
from mcp.server.auth.middleware.auth_context import get_access_token
from mcp.server.auth.provider import AccessToken, TokenVerifier
from mcp.server.auth.settings import AuthSettings
# 下面这些确认门类型都在 mcp.server.mcpserver 里（v2 路径，别写 fastmcp）
from mcp.server.mcpserver import (AcceptedElicitation, CancelledElicitation, DeclinedElicitation,
                                  Elicit, ElicitationResult, Resolve)
RESOURCE = "http://127.0.0.1:8000/mcp"

class RdsTokenVerifier(TokenVerifier):  # 生产里换成 JWT 校验或 RFC 7662 introspection
    async def verify_token(self, token: str) -> AccessToken | None: ...

mcp = MCPServer(
    "ops-readonly",
    token_verifier=RdsTokenVerifier(),  # 硬规则：与 auth= 成对出现，只传一个直接 ValueError
    auth=AuthSettings(issuer_url=AnyHttpUrl("https://auth.corp.example.com"),
                      resource_server_url=AnyHttpUrl(RESOURCE),
                      required_scopes=["ops:read"], validate_token_resource=True),
)

class Confirm(BaseModel):
    ok: bool

async def confirm_close(ticket_id: str) -> Confirm | Elicit[Confirm]:
    """确认门 resolver：只在真要写的时候问人，直接返回 Confirm 就省一次往返。"""
    if ticket_id.startswith("auto-"):
        return Confirm(ok=True)
    return Elicit(f"确认关闭工单 {ticket_id}？会写库并发通知。", Confirm)

@mcp.tool()
async def close_ticket(ticket_id: str,
                       confirm: Annotated[ElicitationResult[Confirm], Resolve(confirm_close)]) -> str:
    """关闭工单（写操作）。硬规则：隐藏工具不算安全，执行层必须自己判 scope。"""
    tok = get_access_token()
    if tok is None or "ops:write" not in (tok.scopes or []):
        # 模型再聪明也不该绕过鉴权 → 协议层错误，不给它重试的机会
        raise MCPError(-32602, "调用方缺少 ops:write scope")
    match confirm:  # "没执行"有四种语义，分开返回，别统一成错误
        case AcceptedElicitation(data=Confirm(ok=True)):
            return f"closed {ticket_id}"
        case AcceptedElicitation():
            return "未执行：确认结果为否"
        case DeclinedElicitation():
            return "未执行：用户拒绝"
        case CancelledElicitation():
            return "未执行：用户取消"
```

**今天的动作**

- [ ] 给 server 加 `TokenVerifier` + `AuthSettings`，用 `curl` 裸调 `/mcp` 验证拿到 401 且 `WWW-Authenticate` 指向 `/.well-known/oauth-protected-resource/mcp`。
- [ ] 在 handler 里用 `get_access_token().scopes` 做 scope 判定，并写清该用 `ToolError` 还是 `MCPError`（判定句："更聪明的模型能避免这次失败吗？"）。
- [ ] 给客户端调用包一层 `asyncio.timeout`（协议没有 per-call timeout，deadline 值与理由写进注释）；`is_error` 分支不要 `raise` 打断编排，要交回模型。同时给自己建的 `httpx2.AsyncClient` 显式设 `timeout=`。
- [ ] 挑一个有副作用的工具加确认门（resolver 或应用层审批），并验证"拒绝"分支返回的是"未执行"而不是错误。
- [ ] 输出 `docs/lessons/w18-governance-rules.md`：把上面五条写成团队能照做的规则清单。git commit。

**会踩的坑**

1. **以为加了 verifier 就全局安全**：stdio 与进程内 `Client(mcp)` 完全不经过鉴权中间件（`get_access_token()` 恒为 `None`）。本地测试全绿、上线后越权，就是这么来的。
2. **把业务失败 `return` 成字符串**。`is_error=False`，模型与宿主 UI 都认为工具成功了、那串文本就是答案。永远 `raise ToolError`，标志位才是信号。
3. **崩溃错误原文回给模型**。SDK 默认不外泄 traceback，但如果你在 `ToolError` 里 `str(exc)` 拼了 SQL 或内部路径，等于自己泄露内部结构。
4. **`json_response=True` 之后确认门失效**。单条 JSON 响应没有反向通道，工具中途要问用户会抛 `NoBackChannelError`，与本次调用绑定的进度通知也会被丢弃。要么用 resolver，要么别开这个开关。

**自测**：① 协议层错误与内容层失败分别用什么异常？判据是哪一句话？② 为什么"按角色隐藏工具"不能当安全控制？正确组合是什么？③ stdio server 的信任边界到底是什么？该在哪一层补上访问控制？

## 周五 · 换成 MCP 调用：跟直连函数算一笔账

**本质**：把工具层从"import 一个函数"换成"通过 MCP 调自己的 server"，本质是用**进程边界换复用性与解耦**。这笔账只有在你真的有两个以上宿主时才划算。

**Java 视角**：直连函数 = 方法调用；MCP = 把内部能力抽成一个独立服务并给它一份标准 IDL。你会立刻问："那我为什么不走普通 REST？"——差别就在 `tools/list` 返回的是模型可用的 schema，宿主能自动接上，你不用给每个宿主写一套适配。

**核心讲解**

要替换的三处：① 把 server 侧的工具定义搬到 `mcp_servers/`，进程内函数只留实现；② Agent 侧启动一个长连接（stdio 子进程或 HTTP）并 `list_tools()`，把返回的 `name/description/input_schema` 直接映射成宿主 SDK 的 tool 定义；③ 工具调用的分派从"查字典调函数"变成"`call_tool` + 解读 `content/structured_content/is_error`"。

必须诚实记录的代价：多一次序列化与进程/网络跳转（本地 stdio 通常 1–10ms 量级，HTTP 跨网络更多，取决于你的部署）、多一个失败点（server 没起来、版本不匹配、schema 协商差异）、调试时"错误在哪一层"变复杂。收益：工具定义只有一份、能被别的团队与别的宿主复用、宿主换模型不用改工具代码、权限与审计能落在 server 层统一做。**判据：如果这个能力只有你自己的 Agent 用、且延迟敏感，直连函数更好。**

**代码**

```python
from __future__ import annotations
from dataclasses import dataclass
from mcp import Client, StdioServerParameters
from mcp.types import Tool
@dataclass(slots=True)
class McpToolBridge:
    """把 MCP server 的工具桥接成宿主 SDK 的 tool 定义 + 统一调用出口。"""
    client: Client
    tools: list[Tool]

    @classmethod
    async def connect(cls, params: StdioServerParameters | str) -> "McpToolBridge":
        # 生命周期由调用方维持（用 async with，或配对的 __aexit__）：别在这里返回一个"已断开"的 client
        client = Client(params)
        await client.__aenter__()
        return cls(client=client, tools=(await client.list_tools()).tools)

    def openai_tool_specs(self) -> list[dict[str, object]]:
        # name / description / input_schema 就是全部契约；空描述是模型用错工具的第一大来源
        return [{"type": "function", "function": {"name": t.name, "description": t.description or t.name,
                                                  "parameters": t.input_schema}} for t in self.tools]

    async def call(self, name: str, args: dict[str, object]) -> dict[str, object]:
        r = await self.client.call_tool(name, args)
        if r.is_error:  # 失败交回对话：让模型看到并可能自我纠正
            return {"ok": False, "for_model": str(r.content)[:1000]}
        return {"ok": True, "data": r.structured_content}  # 成功交回程序：走结构化通道
```

**今天的动作**

- [ ] 用 `McpToolBridge` 把项目二的工具层接到自己的 server 上，跑通同一条链路（W17 的 supervisor + 2 worker）。
- [ ] 同一批 5 个任务分别用"直连函数"与"MCP 调用"各跑 3 遍，记录端到端延迟中位数、错误率、你排查一个问题需要看几处代码。
- [ ] 把 `is_error` 分支与 `structured_content` 分支分别打通：确认业务失败会回给模型、成功数据走结构化通道。
- [ ] 明确写出边界结论：哪些能力留在 MCP（跨团队复用/需要审计），哪些回退直连（本地、延迟敏感）。git commit。

**会踩的坑**

1. **每次调用都重连 server**。stdio 形态下每连一次都要拉起子进程（几十到几百毫秒），表现为"加了 MCP 后延迟翻倍"。诊断：打印连接建立耗时。修法：进程级复用长连接。
2. **把 `content` 里的文本再解析一次**。你已经有 `structured_content`，别用正则去啃模型可读文本。
3. **工具集重复定义**。server 里一份、Agent 里一份，改一处忘另一处，模型看到的 schema 与服务端不一致，报错在运行期才出现。诊断：把两侧的工具名集合做一次 `assert`。
4. **server 起不来时 Agent 全线崩**。今天先把失败处理写成显式的 `BridgeConnectError`（W20 周二会补成测试）。

**自测**：① MCP 相比直连函数多花的钱具体花在哪三处？② 什么判据下你应该回退直连？③ 桥接层为什么不能让模型看到 `content` 之外的实现细节？

## 周六 · 深度块（120 分钟）

### 第一段 55' · 端到端联调

- [ ] supervisor + 2 worker，其中至少 1 个 worker 的工具来自你自建的 MCP server（这就是阶段 4 的验收项）。
- [ ] 跑四类路径：正常、工具描述歧义导致选错、server 未启动、上游超时。逐条记录现象与你给的错误文本。
- [ ] 打开治理层再跑一次带鉴权的调用（HTTP 形态），确认 401 → 元数据 → 带 token 重试这条路真的能走通。
- [ ] 录演示，存 `projects/p2-workflow/demo/`。

### 休息 10'（离开屏幕）

### 第二段 55' · 写《MCP 解决了什么、没解决什么》

诚实写，不要写成推广文。四条局限必须自己测出数字，不接受"听说"：

- [ ] **延迟叠加**：宿主→模型→MCP→你的实现→上游，每层都有开销；测出你这条链路里 MCP 那一跳的净增量。
- [ ] **上下文膨胀**：`tools/list` 的所有 name + description + inputSchema 都要进上下文。数一下每个工具占多少 token，乘以工具数，和检索/历史预算对比。占比高到离谱时，结论是"少注册工具"而不是"换更大模型"。
- [ ] **选择准确率随工具数下降**：把工具数从 3 加到 12，同一批 10 个问题跑一遍，记录选对工具的比例。你多半会看到非线性下滑，这就是真实结论。
- [ ] **多租户鉴权仍要自己做**：协议给了 OAuth 资源服务器的骨架，但"这个令牌能不能查这张表"是你的业务判断，`get_access_token().scopes` 之后那一步没人替你写。
- [ ] 补一条你自己的：调试体验（跨进程 traceback、`mcp dev` 与真实宿主行为不一致）、版本演进冲击（v1 教程失效、roots/sampling/logging 在 2026-07-28 被废弃）、危险操作确认依赖宿主是否真的尊重 annotations。
- [ ] README 更新 + 本周状态（完成 / 部分 / 未做）。

## 本周术语卡

| 术语 | 一句话解释 | 为什么重要 |
|---|---|---|
| MCP / Model Context Protocol | 基于 JSON-RPC 2.0 的模型上下文与能力标准协议 | 把 M×N 私有对接变成 N 次标准实现 |
| host / client / server | 宿主应用 / 一个连接会话 / 能力提供方 | 出问题时先问"是哪一层" |
| capability negotiation | 初始化时双方声明支持哪些原语 | 没声明的能力调了就是错 |
| tools | 模型决定调用的动作 | 描述质量决定模型用不用得对 |
| resources | 应用决定加载的只读上下文，按 **URI** 寻址；`users://{id}/profile` 这类模板进 `resources/templates/list` | 与 tool 的区别是控制权，不是"只读" |
| prompts | 用户从菜单选的消息模板，参数是扁平字符串 | 不参与模型决策，别当提示词管理用 |
| stdio / Streamable HTTP | 子进程管道 / 可部署的 HTTP 传输 | SSE 已被取代；stdio 没有鉴权头 |
| ToolAnnotations | `read_only_hint` 等行为提示 | 规范规定不可信，不是安全控制 |
| isError（内容层）vs JSON-RPC error（协议层） | 前者模型能读到并自我纠正，后者模型看不到 | `ToolError` 与 `MCPError` 的选择依据 |
| structured_content | 工具返回值给程序看的 JSON 通道 | 别再解析 `content` 文本 |
| elicitation / resolver | 工具执行中途问用户的能力 | 危险操作确认门的标准做法 |

## 延伸阅读（可选）

- MCP 官方规范 [modelcontextprotocol.io/specification/2026-07-28](https://modelcontextprotocol.io/specification/2026-07-28) 的 architecture / transports / tools 三页，本周只读这三页就够；同一站点的 `…/2026-07-28/changelog` 是这版改了什么的最权威一份，别拿它当更新日志跳过。Python SDK v2 看仓库 [modelcontextprotocol/python-sdk](https://github.com/modelcontextprotocol/python-sdk) 的 README 与迁移说明，能帮你把网上 v1 教程自动翻译成 v2 写法。
- 发布说明 [blog.modelcontextprotocol.io/posts/2026-07-28/](https://blog.modelcontextprotocol.io/posts/2026-07-28/)：为什么这一版要把会话和握手拆掉（无状态、可缓存、可路由），读一遍你就有了"为什么 MCP 往这个方向走"的判断材料——面试聊协议设计时这是能加分的一段。
