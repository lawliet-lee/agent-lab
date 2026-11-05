# W23 · 作品集与 Java 差异化

> 本周在计划中的位置：阶段 5 收口。前 22 周你把两个项目做完了，但"做完"和"别人能看见、能验证、能给出评价"之间还隔着一层——本周就是把这层补上：一篇公开技术文、一个作品集整页、一只脚踏进 Java 智能体生态（Spring AI / LangChain4j）。不学新原理，全部是"把已有工作变成对外资产"。本周主线一句话：**写"为什么不这样做"比写"我做了什么"更值钱**——周一把它展开成可操作方法，之后每天都复用。

## 本周导航

| 日 | 主题 | 今天的产出物 |
|---|---|---|
| 周一 | 两个项目 README 精修：把"为什么不这样做"写出来 | 两份升级后的 README + `notes/w23-readme-tradeoffs.md` |
| 周二 | 公开技术文：从实验笔记到成稿 | `articles/` 草稿（三选题其一） |
| 周三 | Java 差异化第一步：Spring AI 最小 demo | `java-lab/` 可跑 demo + `notes/w23-jdk-setup.md` |
| 周四 | 项目一能力 Java 重写 + 双栈对照 | `java-lab/` 校验器模块 + `notes/w23-python-vs-java-agent.md` |
| 周五 | 简历改写：把后端经历翻译成 AI 工程底座 | 简历 v2 + 每个数字的证据链检查表 |
| 周六 | 作品集整页：GitHub 门面 + 归位 | profile README、Pinned、成稿文章发布 |
| 周日 | 休息，不许补课 | — |


## 周一 · README：门面即判断力

**本质**：README 的职责不是解释项目是什么，而是让陌生评审者（面试官、潜在协作者）在三分钟内知道该问什么问题、在哪一步不用替你操心。他不关心你实现了 X，关心你在 A 和 B 之间的判断质量。所以最能拉开差距的一节是"为什么不这样做"：做了什么，模型也能替你复述；拒绝了什么、依据什么、什么条件下会改主意，只有真做过的人写得出。

**Java 视角**：这就是 ADR（Architecture Decision Records）换了个载体。企业内网你会写"消息队列为什么选 Kafka 不选 RabbitMQ"；开源作品集没有 wiki，README 就是唯一的 ADR 存放处。已知问题清单则对应上线前的 known-issues 评审——不是自我批评，是降噪。

**核心讲解**

README 精修三件套：
1. **一句话定位**：解决什么问题 + 给谁用 + 规模边界（"单用户 CLI 演示，非服务化"这种边界必须写，否则读者按生产系统来评审你）。
2. **架构图**：不用画漂亮，ASCII 节点图就够，但必须标出校验门、中断点和 trace 落盘位置——能被问"一次请求在哪步会失败"的图才是架构图。
3. **演示 GIF/视频**：30–60 秒终端录屏，剪掉 pip install，直达结论输出。

"为什么不这样做"的操作方法是三问模板，每条决策三问缺一不可：
- 我拒绝了哪个"显然的做法"？（写成问句：为什么没用向量库？为什么不用 LangChain 全家桶？）
- 拒绝的依据是什么？必须指向一次实测、一份 trace 或一条硬约束，"感觉不方便"不算依据。
- 什么条件下我会改？写出触发事件（"评测集超过 500 条且召回延迟 > 800ms 时引入 Qdrant"）。这一行把"偷懒"和"判断"区分开，也是面试反问环节的弹药。

已知问题清单 ≥5 条：你写出来的问题替你挡掉一半负面反馈；没写出来的，会被当成"这人没发现问题"。

**代码**

````markdown
<!-- README 模板骨架：两个项目各按此升级，缺哪节补哪节 -->
# 项目名
> 一句话定位：给【谁】解决【什么问题】，边界是【什么不是】。

## 演示
（30–60s 录屏/GIF，直达一次真实问答的完整过程）

## 架构
（ASCII 图：节点 + 条件边 + 校验门 + trace 落盘点）

## 快速开始
三步：clone → 配 .env.example → 一条演示命令（逐字验证过能跑）

## 为什么不这样做        ← 本周主战场
### 为什么没用向量数据库？
- 拒绝的显然做法：…
- 依据：实测/trace/硬约束，指到具体文件
- 改主意的触发条件：…
（≥2 条这样的决策）

## 已知问题与取舍
（≥5 条，每条一行：现象 + 为什么不修 + 修的成本）
````

**今天的动作**

- [ ] 项目一 README 按模板升级：补"为什么不这样做"≥2 条决策（每条过三问模板）、已知问题 ≥5 条、重画架构图标出校验门。
- [ ] 项目二 README 同样处理，一句话定位用"给谁+解决什么+边界"三段式重写，附演示录屏链接。
- [ ] 以陌生人身份通读两份 README，写下你会问的 3 个问题；答不出答案的，就是把答案补进 README 的任务。
- [ ] 被拒绝的"显然做法"和触发条件整理进 `notes/w23-readme-tradeoffs.md`——周五简历、周六作品集直接复用。git commit。

**会踩的坑**

1. **取舍节写成夸耀节**。表现："考虑了 X，最终选择了更优雅的 Y"。诊断：每条决策里找"约束/实测/触发条件"三要素，缺一即是在自夸。修法：把形容词删光，换成指向 `evals/reports/` 或某次实验文件的引用。
2. **架构图是装饰**。表现：只有方框箭头没有标注。诊断：让别人只看图回答"一次请求在哪步失败"，答不出即中招。修法：标出校验门、interrupt 点、trace 路径三处。
3. **演示超过 60 秒**。表现：含安装等待、含滚动日志。诊断：信息密度不足，陌生人 30 秒内看不到结论就关页面。修法：加速后期剪辑或重录，砍到 30–60 秒。
4. **快速开始三步跑不通**。表现：别人 clone 后卡在第一屏报错就放弃了。诊断：在全新目录逐字照 README 跑一遍。修法：`.env.example` + 安装 + 演示命令写死在 README，跑通一次截图存档。

**自测**：① 你 README 里哪条"为什么不这样做"能答出触发条件？答不出的那条还算取舍吗？② 已知问题 5 条和 0 条，评审者对哪个项目更放心？为什么？③ 你的一句话定位三段式缺哪段？


## 周二 · 技术文：实验笔记不是教程

**本质**：公开技术文的价值不是科普——"LangChain 五步入门"这类文章模型写得比你快，差异化价值为零。陌生人不可替代的只有你的实验设计、数据、失败尝试和结论边界。写作标准一句话：删光代码块和步骤截图后，剩下的判断撑不起一屏，就是教程不是文章。

**Java 视角**：对应内网技术复盘，但读者不能追问你背景，所以结论必须前置到前三屏；也不能拍你细节，所以反驳要自己写——主动写"这个结论在 X 数据量、Y 模型下不成立"，比被人评论区挑穿体面得多。

**核心讲解**

素材三选一，别贪：
1. **W14 检索对比实验**（切分策略/嵌入模型 vs hit-rate）——最硬，自带数字和评测集，首推。
2. **MCP 落地局限**（`experiments/w18-what-mcp-does-not-solve.md` 已有底稿）——有一个新鲜的时效论据：MCP 规范 2026-07-28 版移除了 initialize 握手与协议级会话（`Mcp-Session-Id`），而 Spring AI 2.0.2 的 release notes 里已经出现 "Add customizers for stateless MCP servers"——Java 生态正在跟上无状态化，说明协议层共识在收敛，你 W18 记的局限里哪些已被解决、哪些还在，正好是文章骨架。
3. **W17 多智能体拆分判断**——如果你那张对比表的结论是"多 Agent 更差"，这是最稀缺的选题，因为逆着宣传声量且有实测数据。

"从实验笔记到成稿"结构模板：
- 开头三段：业务背景一句话 → 问题一句话 → 结论一句话（别卖关子）。
- 实验设计：输入集、评测集条数、判定标准，读者能照做。
- 结果：表格，每行一句话点评。
- 意外与失败尝试：不许删，这是文章的正文，教程化写法第一凶手就是只留成功路径。
- 结论边界：什么数据量/模型/领域下结论不成立，主动写。

**代码**

````markdown
<!-- articles/w23-draft.md 骨架：周二填内容，周六发布 -->
# 标题（写结论或反直觉点，不写"初探/浅析/实践"）

## TL;DR
三句话：场景 → 问题 → 结论（含边界）。

## 实验设计
评测集 N 条，判定标准 …，数据见 `evals/reports/…`（相对路径，读者可 clone 复现）。

## 结果
| 配置 | 指标 | 一句话点评 |

## 失败的尝试（正文，不是附录）
试了 X 不行 → 从 trace 看到 … → 归因 …

## 结论的边界
本结论在 … 下不成立；下一步 …。
````

**今天的动作**

- [ ] 定选题（30 分钟内，用"哪个有现成数字"判，别用"哪个更有趣"判），把实验笔记原文复制进 `articles/` 草稿旁对照。
- [ ] 先写前 400 字（背景+问题+结论），发给一位懂技术的朋友，问"这篇文章值不值得读完、凭什么信你"，把回复原话记下来。
- [ ] 按模板成稿，重点补"失败的尝试"节：至少 3 处，每处有证据文件。全篇数字逐个过"来源=仓库文件"检查。
- [ ] 成稿不发布，存 `articles/w23-*.md`，周六门面弄完再发。git commit。

**会踩的坑**

1. **开头铺垫两屏**。表现：第一段介绍什么是 LLM。诊断：目标读者滑两屏还没见到结论。修法：TL;DR 前置，背景压到一句。
2. **数字没有出处**。表现："准确率明显提升"。诊断：这个数指得到哪个文件？指不到就是凭印象，凭印象就是埋雷。修法：每个数字带 `evals/reports/` 路径，示例路径一律相对路径，公开仓库别暴露本机目录。
3. **把工具文档转述当内容**。表现：通篇 API 示例、没有一次失败。诊断：删掉代码块还剩多少判断。修法：强制"失败的尝试"一节 ≥3 处。
4. **追求大而全**。表现：想把 W14 到 W18 全装进一篇。诊断：写到 3000 字收不住。修法：一篇只回答一个问题，剩下的素材留给下一篇或 README。

**自测**：① 你文章里哪一句会被读者反驳？一句都没有说明写了常识。② 全文最大的数字，出处文件路径说出来。③ 三个候选选题里你放弃的那两个，理由是什么？


## 周三 · Spring AI 最小 demo：先装 JDK 再谈生态

**本质**：Java 差异化的第一道门槛不是框架 API，是工具链。本机 JDK 1.8.0_441 连门都进不了——Spring AI 2.0.x 只支持 Spring Boot 4.0.x 与 4.1.x，而 Boot 4 的 Java 基线是 17（Spring Framework 7 同样 17+）。今天的产出物不是"理解了框架"，是"本地环境能起 Boot 4 + 模型调通一次"的闭环。

**Java 视角**：终于回到主场，但换个方向体会 Python 侧的脆弱：agent-lab 的 3.12 解释器 + pyproject 随便换台机器就可能起不来；Java 这边依赖治理成熟得多（BOM 统一版本、坐标全局唯一），代价是版本组合是硬法律——Boot 4 ↔ Spring AI 2.0.x ↔ JDK 17+ 一条都不能错，错了不是警告是起不来。

**核心讲解**

版本事实（2026-11-05 经官方渠道核实）：Spring AI 最新 **2.0.2**（2026-10-23 发布），官方 getting-started 原文 "Spring AI 2.0.x supports Spring Boot 4.0.x and 4.1.x"；Boot 4.0 的 Java 基线 **17**。LangChain4j 最新 **1.23.0**（2026-10-28）。

JDK 安装策略：**最低 17 才能跑 Boot 4，但建议直接装 21 LTS（当前主流）**。别替换系统默认 1.8——你手头的企业老项目还要它，macOS 上用 `JAVA_HOME` 切换或 sdkman/jenv 做多版本并存，切换方法记进笔记（这本身就是周四对照笔记的素材：Java 的多版本共存是一等公民，Python 的 venv 解决依赖却很少操心解释器并存）。

依赖形态：BOM 进 `<dependencyManagement>`（`<type>pom</type><scope>import</scope>`），starter 用 `spring-ai-starter-model-openai` / `spring-ai-starter-model-anthropic`，业务依赖不写版本号，全由 BOM 管。密钥永远从环境变量读，`application.yml` 里 `${OPENAI_API_KEY}` 占位，**不写默认值**——写了默认值就有 temptation 提交真值。

**代码**

```xml
<!-- pom.xml 骨架：Spring AI 2.0.2 为已核实版本；Boot 具体 patch 号以
     start.spring.io 生成结果为准（版本链硬约束：Boot 4.0.x/4.1.x + JDK 17+） -->
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>4.0.x</version>
    <relativePath/>
</parent>
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.ai</groupId>
            <artifactId>spring-ai-bom</artifactId>
            <version>2.0.2</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
<dependencies>
    <dependency>
        <groupId>org.springframework.ai</groupId>
        <artifactId>spring-ai-starter-model-openai</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>
```

```yaml
# src/main/resources/application.yml（配置键名以官方文档当前写法为准）
# 密钥只从环境变量读，不写默认值，不进 git
spring:
  ai:
    openai:
      api-key: ${OPENAI_API_KEY}
```

```java
// 最小调用。ChatClient.Builder 注入 + prompt().user(...).call().content()
// 已对照 docs.spring.io/spring-ai/reference/api/chatclient.html（2.0.2 文档）核实
@RestController
class AskController {
    private final ChatClient chatClient;

    AskController(ChatClient.Builder builder) {
        this.chatClient = builder.build();
    }

    @GetMapping("/ask")
    String ask(@RequestParam String q) {
        return chatClient.prompt().user(q).call().content();
    }
}
```

```java
// LangChain4j 对照（dev.langchain4j:langchain4j-open-ai:1.23.0）：
// 无容器、无自动配置，10 行跑通；API 形态已对照官方文档核实
ChatModel model = OpenAiChatModel.builder()
        .apiKey(System.getenv("OPENAI_API_KEY"))
        .modelName("gpt-4o-mini")   // 具体模型名以你的供应商为准
        .build();
String answer = model.chat("用一句话解释 outbox 模式的失败场景。");
```

两个形态的差异今天就记：Spring AI 把模型实例挂进 Spring 上下文、配置走 yml、自动配置接管——适合嵌入既有服务；LangChain4j 不依赖容器、builder 手搓——适合小脚本和非 Boot 服务。没有优劣，只有你的目标场景。

**今天的动作**

- [ ] 装 JDK 21（1.8 保留不动），`java -version` 与 `mvn -version` 分别确认，切换方法写进 `notes/w23-jdk-setup.md`。
- [ ] 用 start.spring.io 生成骨架（Boot 4.0.x + OpenAI starter）或按上面 pom 手搓，`mvn spring-boot:run` 起来，`/ask` 返回模型回复。
- [ ] 跑 LangChain4j 对照片段（一个 main 方法即可），记录两条路径从 clone 到第一次调通的耗时与卡点。
- [ ] 到此为止——流式、多模型路由、工具调用全部不许今天碰，写进 backlog。git commit。

**会踩的坑**

1. **`invalid target release` / class file version 报错**。表现：mvn 编译直接失败。诊断：`mvn -version` 打印的仍是 1.8——终端 JAVA_HOME 和 IDE 的 Project SDK 是两个开关，常只改了一个。修法：两处都指到 21，用 sdkman/jenv 管理。
2. **Boot 与 Spring AI 组合错**。表现：启动时 NoSuchMethodError、依赖树版本冲突。诊断：`mvn dependency:tree` 找 spring-boot 与 spring-ai 版本。修法：记住硬约束 2.0.x ↔ Boot 4.0.x/4.1.x，别试图在 Boot 3.x 上硬塞。
3. **密钥进 git**。表现：`application.yml` 里写了真实 key 或默认值。诊断：提交前 grep 一次 `sk-`。修法：`${OPENAI_API_KEY}` 占位 + 本地覆盖文件进 `.gitignore`。
4. **把 demo 当生产搭**。表现：天黑了还在加 SSE 流式和多模型路由。诊断：60 分钟线早过了。修法：今天验收标准就一条——一个端点一次调用，其余进周四/周六。

**自测**：① 说出 Spring AI 的版本链三层约束；Boot 3.x 配 Spring AI 2.0.x 会发生什么？② 同一个 `ChatModel` 实例的生命周期，Spring AI 和 LangChain4j 各归谁管？③ 密钥为什么不能写成 `${OPENAI_API_KEY:sk-xxx}`（带默认值）形式？


## 周四 · 把项目一的一个能力搬进 Java：对照笔记才是产出物

**本质**：周三是"跑通别人的样例"，今天是"移植一件自己的东西"。选项目一里最确定的一块——**结构化输出校验器**（纯逻辑、无网络、有现成失败用例）——重写成可嵌入现有服务的模块：接口 + 实现 + 单测。但记住：demo 是载体，今天真正的产出物是那篇 Python/Java 双栈对照笔记，它是你面试里"技术选型"环节唯一有实据的发言。

**Java 视角**：定义接口、实现、注册 Bean、JUnit 兜底——这是你的日常动作，闭着眼都会。有意思的是反向记录差异：pydantic v2 的 `model_validate` + `field_validator` vs Jackson 反序列化 + 手写校验；pytest 函数 vs JUnit 5 类；错误列表回喂（W12 的 `REPAIR_PROMPT`）在 Java 侧怎么表达。

**核心讲解**

对照笔记三轴，每轴要落到你自己的代码/文件，不许写"Java 稳定、Python 灵活"级别的空话：
1. **生态成熟度**：LLM 生态的第一现场在 Python——新模型、新协议、评测工具先在 Python 出现，Java 侧是跟进移植。差距在缩小但有滞后：Spring AI 2.0.2 已收录 stateless MCP server customizers（呼应周二说的规范 2026-07-28 无状态化），LangChain4j 1.23.0 的检索与工具抽象也齐了。判据不是"谁星多"，是**你依赖的具体能力在哪边什么时候出现的**——本周你已实测两个框架的接入路径，写你的滞后体感。
2. **可维护性**：Java 的类型系统、静态契约、IDE 重构是长期多人维护的真实红利；Python 靠 pydantic 把契约拉齐，但 dict 满天飞处类型失效，重构要靠测试网兜底——你有单测，所以能对比"改一个字段名两边各要动几处、编译/测试各报不报"。
3. **企业落地成本**：这是你手里最大的牌。企业要把 AI 能力嵌进既有 Java 服务，而不是新起 Python 服务再跨语言 HTTP。同时一个国内现实：**大量存量服务还跑在 JDK 8**，Boot 4 要 17+，意味着 Spring AI 这条线对多数存量系统并不现实——要么停在旧 Boot + 旧 Spring AI 线，要么走 LangChain4j（基线低，能进 Boot 2/3 甚至裸 Java 服务）。面试时能把这个落差讲清楚（"为什么我们公司大部分服务不会直接上 Spring AI"），比讲框架 API 值钱得多，因为它是带业务约束的取舍。

**代码**

```java
// java-lab/src/main/java/com/agentlab/validator/ 独立模块，接口先行，不放 controller 里
public interface FinalAnswerValidator {
    List<String> validate(String rawJson, Set<Integer> knownLines);
}

public record FinalReport(String rootCause, List<Integer> evidenceLines, String nextAction) {}

public class ReportValidator implements FinalAnswerValidator {
    private final ObjectMapper mapper = new ObjectMapper();

    @Override
    public List<String> validate(String rawJson, Set<Integer> knownLines) {
        FinalReport report;
        try {
            report = mapper.readValue(rawJson, FinalReport.class);
        } catch (JsonProcessingException e) {
            // Jackson 版本随 Boot 4 依赖树确定，record 反序列化若字段名对不上先查它
            return List.of("输出不符合契约: " + e.getOriginalMessage());
        }
        List<String> errors = new ArrayList<>();
        if (report.evidenceLines() == null || report.evidenceLines().isEmpty()) {
            errors.add("证据行不能为空");
        } else {
            report.evidenceLines().stream()
                  .filter(n -> !knownLines.contains(n))
                  .forEach(n -> errors.add("引用了不存在的日志行 L" + n));
        }
        return errors;
    }
}

class ReportValidatorTest {
    @Test
    void rejectsFakeEvidenceLine() {
        List<String> errors = new ReportValidator().validate(
                """
                {"rootCause":"连接池耗尽","evidenceLines":[7],"nextAction":"扩容"}""",
                Set.of(1, 2, 3));
        assertTrue(errors.stream().anyMatch(e -> e.contains("L7")));
    }

    @Test
    void emptyEvidenceRejected() {
        List<String> errors = new ReportValidator().validate(
                """
                {"rootCause":"连接池耗尽","evidenceLines":[],"nextAction":"扩容"}""",
                Set.of(1, 2, 3));
        assertTrue(errors.stream().anyMatch(e -> e.contains("证据行")));
    }
}
```

对照 W12 的 pydantic 版记三件事：返回错误列表 = 回喂 `REPAIR_PROMPT` 的原料，两边语义要逐字对齐；pydantic 里 `field_validator` 挂契约上，这里校验和反序列化分开（Jackson 只管形状，业务规则自己写）；"引用行号 ∈ 工具真实返回"这条防线哪边写得便宜——数行数，数字进笔记。

**今天的动作**

- [ ] 建 `java-lab/.../validator/` 模块：接口 + 实现 + 上面两个单测先跑绿（`mvn test`）。
- [ ] 用 Python 版的两个历史坏用例（缺字段/编造行号）逐字验证 Java 版行为一致，尤其错误文案能否直接拼进回喂提示。
- [ ] 写 `notes/w23-python-vs-java-agent.md`：三轴 × 每轴 2–3 条实测点（指到具体文件和行数），必须含 JDK 8 存量落差一段。
- [ ] git commit。

**会踩的坑**

1. **把重写做成语法翻译**。表现：写完了说不出两边差异。诊断：笔记里没有一条"只在一边存在"的点就是在翻译。修法：每条轴强制配一个具体代码对照。
2. **JSON 边界翻车**。表现：record 字段映射不上、snake_case 全 null。诊断：异常信息或空对象。修法：命名策略配置或注解；把它记进笔记——"类型系统的严谨在边界处的成本"正是对照素材。
3. **scope 膨胀到 VectorStore**。表现：深夜骨架还没齐。诊断：手伸向了网络调用。修法：今天只碰确定性模块，检索版留 backlog。
4. **笔记写成营销文**。表现："Java 稳健、Python 灵活"。诊断：逐条问"证据在哪一行"，指不出即空话。修法：删掉或换成实测。

**自测**：① 三轴里哪条面试官最常听人吹、而你有一手数据？② JDK 8 落差为什么在面试里是你的优势而不是短板？③ 若换成重写 `retrieve`，接口签名要多什么依赖？为什么它不适合昨天做？


## 周五 · 简历：数字有出处，或者删掉

**本质**：简历改写不是"转行声明"，是**翻译**。五年后端经历和 Agent 工程的核心问题是同一组问题——重试风暴、状态丢失、故障定位、成本控制；24 周的 agent-lab 是你在新领域解决过它们的实证。每条新项目条目三段式：建了什么 + 怎么验证 + 可查的数字。

**Java 视角**：你知道看简历的手感——面试候选人时，"熟悉 LangChain"是背景噪音，"32 条评测集把 hit-rate@5 从 0.62 提到 0.79"会让你追问。按你挑候选人的标准写自己的简历。

**核心讲解**

翻译清单，三条硬对应，每条配一个你亲手的事件：
- **分布式事务/补偿** → supervisor 状态传递、终止条件、约束丢失修复（W17）："事务上下文在跨服务调用中丢失"和"任务约束在跨 Agent 摘要中丢失"是同一族 bug，你的 `AgentMessage`/`HandoffEnvelope` 就是证据。
- **幂等** → 副作用清单、重放防重复执行（W12 重放检查表、W17 `already_executed_effects`、W19）：Agent 二次执行写操作 ≈ 消息重复消费，你防过两次。
- **可观测性** → JSONL trace、从 trace 定位真实故障（W12 验收第④条、W21）：面试官都懂"没监控不敢值班"，你把这套搬到了 LLM 调用链上。

**可验证数字纪律**（这条是真有用的工程纪律，不是道德说教）：
1. 每个数字必须能指到仓库里的文件——`evals/reports/`、`experiments/` 表格、git log。例："hit-rate@5 从 0.62 提到 0.79，评测集 32 条，见 `evals/reports/`"。面试不知道哪个数字被追问，被追问时打不开文件，整段经历跟着存疑。
2. 写不出来源的数字，删掉或降级成定性描述。"明显提升"不是谎言但也不是优势；编一个精确数字才是雷。
3. 四件套自检：对每个"从 X 到 Y"，当场说出评测集条数、判定标准、前后配置差异、报告文件路径。四件齐才许上简历。

新项目条目模板：一句话定位 → 架构亮点（校验门/终止条件/评测闭环挑 1–2 个）→ 可验证数字 → 仓库与文章链接。

**代码**

```markdown
<!-- 简历条目：左边是改前（工具罗列+无出处），右边是改后 -->
- ❌ 熟悉 LangChain/LangGraph/RAG，参与多个 AI 应用开发，显著提升回答质量。
- ✅ RAG 知识库问答助手（agent-lab，Python 3.12 + LangGraph）
  - 手写 ReAct 循环后迁移到状态机，能说出两边各自代价（见迁移笔记）；
  - 评测集 32 条，两轮切分策略将 hit-rate@5 从 0.62 提到 0.79（evals/reports/）；
  - 每步执行落 JSONL trace，据此定位过一次"工具节点吞异常"的真实故障。
```

**今天的动作**

- [ ] 一张纸列五年经历里可翻译的项（幂等/分布式事务/监控告警/容量规划各对应哪个 Agent 问题），选 3 条并各配一个具体事件。
- [ ] 按模板写两个新项目条目，每个数字后先写出处文件路径再写进正文；写不出出处的当场删除或降级为定性。
- [ ] 全部数字过一遍四件套自检，找一个懂技术的朋友现场抽问 3 个数字，答不上来的回炉。
- [ ] 技能罗列段瘦身：删光"熟悉"，留下的框架词每个都能被追问 5 分钟；示例仓库路径用相对路径或 `~/Desktop/AI/agent-lab` 形式。git commit。

**会踩的坑**

1. **工具清单当能力**。表现：一行列 10 个框架，被追问任何 5 分钟即穿帮。诊断：逐个问"能展开讲什么"。修法：砍到 3 个，且每个在经历条目里有证据。
2. **数字被一秒戳穿**。表现："做了很多评测用例"，被问几条、判定标准，现场卡壳。诊断：四件套缺件。修法：缺件数字不上简历。
3. **经历翻译过度**。表现：简历比仓库宏大一个量级。诊断：简历条目逐条对照 README 和报告文件。修法：作品集是简历的证据链，两处矛盾时砍简历侧——评审的人两边都看。
4. **一份简历打所有岗位**。表现：Java 内部 AI 平台岗和模型应用公司收到同一份。修法：做两版——"Java 差异化"版（前置三四）和"Python 全栈"版（前置一二），强调不同但数字出处必须同一套。

**自测**：① 任选简历上最难的一个数字，四件套当场说全。② 三条翻译里哪条你有具体事件，哪条只是概念对齐？③ 两版简历允许不同的和绝不允许不同的分别是什么？


## 周六 · 深度块（120 分钟）

### 第一段 55' · GitHub 主页整理

- [ ] Pinned 四仓：`agent-lab`（学习计划+讲义，主门面）、p1、p2、`java-lab`。逐仓确认 About 一句话定位、topics、README 首屏就是周一升级后的版本。
- [ ] profile README（同名仓库）：方向一句话"Java 后端 5 年 → 智能体工程，24 周公开学习路径"；两项目 + 技术文 + 简历链接各归位；贴 3 个最能打的数字（必须带出处指向）。不粉饰未完成项——"W19 未启动"写出来比藏住可信。
- [ ] 隐私扫描（公开仓库不可逆，这段不许省）：`grep -rn "/Users/\|/home/" .` 扫全仓工作区与 git 历史，清掉本机用户名、真实公司名、内网域名、密钥；命中历史提交的当天处理掉，别留到下周。

### 休息 10'（离开屏幕，这段不许省）

### 第二段 55' · 技术文发布 + 作品集整页

- [ ] 发布周二成稿：掘金/公众号任一即可，仓库 `articles/` 留底稿，README 与 profile 回填文章链接。发布前最后读一遍：TL;DR 三段、每个数字有出处、失败尝试 ≥3 处。
- [ ] 作品集整页（以 profile README 为主体）：2 项目 ×（定位 + 架构图 + 演示链接 + "为什么不这样做"锚点）+ 技术文 + 简历 PDF 下载。然后做 3 分钟自测——从 profile 进入，假装陌生面试官能否拼出"这人是谁、做过什么、判断力如何"；拼不出的缺口直接补在对应位置，缺演示补录屏，缺取舍说明周一那节没做完就回炉。
- [ ] 定下周（W24）复盘周的输入清单：本周所有"部分完成"项原样抄过去，别修饰。learning-log 写本周状态：完成/部分（缺什么）/未做。git commit。

## 本周术语卡

| 术语 | 一句话解释 | 为什么重要 |
|---|---|---|
| "为什么不这样做" | 被拒方案 + 拒绝依据 + 改主意触发条件三件套 | 作品集里最拉开判断力的一节，本周主线 |
| ADR | 架构决策记录 | README 取舍节的 Java 侧同款概念 |
| 已知问题清单 | 主动写出限制与不修理由 | 降噪：挡掉一半试用负反馈 |
| 可验证数字 | 每个数字能指到仓库内文件 | 简历的可信度单位；写不出来源就删 |
| 四件套自检 | 条数/判定标准/前后对照/报告路径 | 数字被追问时的及格线 |
| 版本链 | Boot 4.0.x/4.1.x ↔ Spring AI 2.0.x ↔ JDK 17+ | 周三环境的硬法律，装不上多半是链错 |
| BOM import | spring-ai-bom 进 dependencyManagement 统一管版本 | Java 依赖治理形态，对照 pyproject |
| stateless MCP | 规范 2026-07-28 版移除 initialize 握手与 Mcp-Session-Id | 周二文章论据；Spring AI 2.0.2 已跟进 customizers |
| 存量 JDK 8 落差 | 国内大量服务仍在 8，Boot 4 要求 17+ | 企业落地成本轴的核心论据，面试可讲取舍 |
| 作品集漏斗 | profile → 项目 README → 文章 → 简历 | 3 分钟内拼出你是谁，漏斗断一环全白 |

## 延伸阅读（可选）

- `docs.spring.io/spring-ai/reference/` 的 ChatClient 与 API 章节——注意文档右上角版本切换，看 2.0.x 的；目的不是通读，是校准你 demo 里那十行。
- Spring AI 2.0.2 的 GitHub release notes（"Add customizers for stateless MCP servers" 一条）与 MCP 2026-07-28 规范变更日志，对读——周二写文的两个信源。
- LangChain4j 官方文档（docs.langchain4j.dev）的 AI Services 一节：看完你会明白它和 Spring AI 的抽象边界划在哪，周四笔记的"生态成熟度"轴多一条实据。本周别动手试，范围纪律高于好奇心。
