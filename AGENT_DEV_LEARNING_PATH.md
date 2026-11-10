# AgentScope Java 深度学习路径 —— 面向 35K Agent 开发岗

> **目标**：以 agentscope-java 为教材，从"会用框架"到"能读源码、能讲设计、能造轮子"。
> **周期**：约 10~12 周（每天 1.5~2 小时），压缩冲刺可 8 周。
> **完成标志**：文末「毕业检查清单」全部打勾 + 完成毕业项目。
>
> ⚠️ 本文件是个人学习笔记，建议加入 `.gitignore` 或移出仓库，避免误提交。

---

## 0. 岗位画像：35K Agent 开发岗到底考什么

先明确靶子。高级 Agent 开发岗位的能力模型（按面试权重排序）：

| 能力域 | 具体要求 | 对应本路径 |
|---|---|---|
| Agent 核心原理 | ReAct 循环、function calling 细节、上下文管理、结构化输出 | 阶段 1、2、5 |
| 流式与响应式编程 | SSE/WebSocket、Project Reactor、背压、取消 | 阶段 2、5 |
| 框架源码与设计模式 | 能讲清 middleware 链、SPI 扩展、Builder 等如何实现 | 阶段 3~8 |
| 生产化工程 | 会话持久化、幂等、故障恢复、权限安全、可观测 | 阶段 7、8、10 |
| 多 Agent / 高级架构 | 子代理编排、协议（A2A/MCP/AG-UI）、沙箱隔离 | 阶段 9 |
| 动手能力 | 有一个能跑、能演示的完整项目 | 毕业项目 |

**面试残酷现实**：初级问题（什么是 ReAct）只能保底，**区分度来自"你读过/写过实现吗"**。比如：
- ❌ 平庸回答："ReAct 就是推理和行动交替"
- ✅ 高薪回答："ReAct 循环在我读过的实现里不是 while 循环，而是递归的 Reactor 组合——`executeIteration` → `reasoning` → 决定是否 `acting` → 末尾递归 `executeIteration(iter+1)`，这样天然获得流式与取消语义。同时 `call()` 只是 `streamEvents()` 流上 `filter(AgentResultEvent).takeLast(1)`，保证了两个 API 终态一致。"

本路径所有"核心讲解"都在帮你形成第二类回答。

---

## 学习路线总览

```
阶段1  会跑 + ReAct 理论        ████░░░░░░  1周
阶段2  消息与事件模型           ████░░░░░░  3天
阶段3  工具系统                 █████░░░░  4天
阶段4  模型抽象与 Provider 生态  █████░░░░  4天
阶段5  ReActAgent 主循环(心脏)  ████████░  1.5周
阶段6  Middleware 扩展体系      █████░░░░  3天
阶段7  状态与会话持久化         ██████░░░  5天
阶段8  权限系统与 HITL         ████░░░░░░  3天
阶段9  Harness 高层架构        ████████░  1.5周
阶段10 生产化工程              ██████░░░  1周
毕业项目  Mini Coding Agent    █████████  2周
```

每个阶段固定五段式：**概念 → 核心讲解 → Demo 实操 → 深挖自测 → 面试考点**。

---

## 阶段 1：会跑 + ReAct 理论（1 周）

### 📖 概念
- **ReAct 范式**：Reasoning（模型思考/决定调工具）与 Acting（执行工具、结果回填）交替，直到模型给出最终回答
- **Function Calling 协议**：模型不直接执行代码，而是输出结构化的 `tool_use`（工具名 + JSON 参数），由框架执行后以 `tool_result` 回填上下文
- **流式 vs 阻塞**：`streamEvents()` 返回 `Flux<AgentEvent>`（全生命周期事件流）；`call()` 返回最终 `Msg`
- **Agent 的最小组成**：Model（大脑）+ Toolkit（手）+ sysPrompt（人设）+ AgentState（记忆）

### 🔍 核心讲解
Agent 的最小可用配置只需 5 行 builder。注意 `model("dashscope:qwen-plus")` 这种字符串写法走的是 `ModelRegistry` 自动解析（阶段 4 会拆解它）：

```java
ReActAgent agent = ReActAgent.builder()
        .name("Assistant")
        .sysPrompt("You are a helpful AI assistant.")
        .model("dashscope:qwen-plus")   // ModelRegistry 按 "provider:model" 解析
        .toolkit(new Toolkit())
        .build();
```

**关键认知**：2.0 版本已**移除 `.memory()`**——会话历史由 `AgentState` 承载，通过 `.stateStore()` 持久化（阶段 7）。如果你在别的资料里看到 `.memory(new InMemoryMemory())`，那是过时 API。

### 💻 Demo 实操
1. 跑通 `agentscope-examples/README.md` 的全部 11 个基础例子（按表格顺序），重点是：
   ```bash
   mvn exec:java -pl agentscope-examples/documentation \
     -Dexec.mainClass="io.agentscope.examples.documentation2.quickstart.BasicChatExample"
   ```
2. **自己写**：复制 `BasicChatExample.java` 到自己的包里，改成一个"代码评审助手"（sysPrompt 换成评审人设），跑通。
3. 观察 `ToolCallingExample.java`：注意 `calculate` 工具故意用简陋的 split 实现——**工具的容错处理本身就是 prompt 工程的一部分**（返回错误提示文本而不是抛异常，让模型自我纠正）。

### 🧠 深挖自测
- [ ] 用户问"东京几点了"，画出从 `agent.call()` 到返回的**完整时序**：几次模型调用？中间发生了什么？
- [ ] `call()` 和 `streamEvents()` 返回的东西分别是什么类型？什么场景用哪个？

### 🎤 面试考点
- "讲一下 ReAct 和 Plan-and-Execute 的区别、各自适用场景"（ReAct：逐步决策、适合工具密集交互；P&E：先规划再执行、适合长任务省 token）
- "function calling 中工具的 JSON Schema 是谁生成的、给谁看的？"

---

## 阶段 2：消息与事件模型（3 天）

### 📖 概念
- **消息模型**：`Msg`（角色 + 内容块列表）→ `UserMessage`/`AssistantMessage`/`ToolResultMessage` 等子类
- **内容块（ContentBlock）**：sealed 层级——`TextBlock`、`ThinkingBlock`、`ToolUseBlock`、`ToolResultBlock`、`ImageBlock`…
- **事件模型（AgentEvent）**：约 28 种事件组成 agent 全生命周期的时间线
- **不可变值对象**：record + 值相等，是整个框架的数据底座

### 🔍 核心讲解
**1. 为什么用 sealed？** `ContentBlock.java` 用 `sealed permits ...` 限定消息内容只可能是那 9 种 block。类型系统保证：加新 block 时编译器强制你处理所有 switch/模式匹配分支——**用编译期穷尽性检查代替运行时 if-else 地狱**。这是 Java 17 代数数据类型（ADT）的标准实践，面试聊 Java 新特性时的顶级素材。

**2. 事件时序**（来自 `AgentEventStreamExample` 的 javadoc，背下来）：
```
AGENT_START
  MODEL_CALL_START
    TEXT_BLOCK_START
      TEXT_BLOCK_DELTA   ← 每个 token 一个
    TEXT_BLOCK_END
  MODEL_CALL_END          ← 携带 token 用量
  TOOL_CALL_START         ← 有工具调用时
  TOOL_CALL_END
  TOOL_RESULT_START
    TOOL_RESULT_TEXT_DELTA
  TOOL_RESULT_END         ← SUCCESS / ERROR
  ... (下一轮 MODEL_CALL_START，ReAct 循环)
AGENT_END
```
注意：**一个 ReAct 循环 = 多次 MODEL_CALL**。`replyId` 关联整次调用，`toolCallId` 关联工具调用/结果对——这套关联 ID 设计直接可用于产品端的事件回放。

**3. 事件怎么到达你手里？** 不是全局事件总线，而是每次订阅创建的 `FluxSink`（pull 模式）。对比 Spring 的 `ApplicationEventPublisher` push 模式：Flux 天然带背压和取消语义。

### 💻 Demo 实操
写一个**事件审计器**（自己写，不要抄例子）：订阅 `streamEvents()`，把每个事件按 `event.getType()` 记录到 console，统计：
- 本轮共几次 `MODEL_CALL_START`（= ReAct 迭代次数）
- 总 input/output token（累加 `ModelCallEndEvent.getUsage()`）
- 工具调用成功率（`ToolResultEndEvent.getState()`）

参考骨架（API 已验证）：
```java
agent.streamEvents(userMsg)
    .doOnNext(e -> {
        switch (e.getType()) { /* AgentEventType 枚举分发 */ }
        if (e instanceof ModelCallEndEvent mce && mce.getUsage() != null) {
            totalIn += mce.getUsage().getInputTokens();
            totalOut += mce.getUsage().getOutputTokens();
        }
    })
    .blockLast();
```

### 🧠 深挖自测
- [ ] 打开 `agentscope-core/.../message/ContentBlock.java`，数一数 permits 了哪几种 block，各自的用途
- [ ] `TextBlockDeltaEvent.getDelta()` 和 `Msg.getTextContent()` 的区别是什么？（增量 vs 全量）

### 🎤 面试考点
- "你的 agent 前端如何做流式渲染？" → 用事件流粒度（TEXT_BLOCK_DELTA）回答
- "Java 17 的 sealed interface 在你项目里怎么用的？"

---

## 阶段 3：工具系统（4 天）

### 📖 概念
- **注解驱动工具**：`@Tool` + `@ToolParam`，普通 Java 方法 → LLM 可调用的工具
- **Schema 自动生成**：反射读取方法签名 → JSON Schema（strict 模式递归注入 `additionalProperties:false`）
- **ToolEmitter**：工具执行中向前端推送进度（不进 LLM 上下文）
- **双层防护**：agent 层权限引擎 + toolkit 层 `ToolCallInterceptor` 洋葱链

### 🔍 核心讲解
**1. 注解 → AgentTool 的桥接**：`Toolkit.registerTool(obj)` 扫描 `@Tool` 方法，`ReflectiveFunctionTool`（Adapter 模式）把 `Method` 适配成 `AgentTool` 契约。特殊参数（`ToolEmitter`、`AgentState`）由框架识别并注入——**依赖注入发生在方法签名级别**。

**2. 双洋葱结构**（本框架最精彩的分层之一）：
```
Agent 层:  onAgent → onReasoning → [权限引擎 evaluatePermissions] → onActing → 工具执行
Toolkit 层: ToolExecutor.applyInterceptorChain() → 参数校验 → timeout 装饰 → retry 装饰 → 真正执行
```
为什么分两层？agent 层管"该不该做"（策略），toolkit 层管"怎么做才安全"（参数改写、审计、超时）。**权限判断和执行防护是不同的关注点**——面试讲安全设计时的高分论点。

**3. Null Object 模式**：`NoOpToolEmitter.INSTANCE`。工具不想要进度回调时不需要判空传 null——优雅处理"可选依赖"的经典手法。

### 💻 Demo 实操
1. **带进度的工具**（API 已验证，自己敲一遍）：
```java
public class ProgressTools {
    @Tool(name = "process_data", description = "Process a dataset and report progress")
    public String processData(
            @ToolParam(name = "dataset_name", description = "Name of the dataset")
            String datasetName,
            ToolEmitter emitter) {                      // ← 框架自动注入
        for (int i = 1; i <= 5; i++) {
            emitter.emit(ToolResultBlock.text("Processed " + i * 20 + "%"));  // → ToolProgressEvent
        }
        return "Done: 1000 records";
    }
}
```
2. **错误处理实验**：让工具故意抛异常 vs 返回错误文本，观察模型行为差异（前者中断循环，后者模型会自我纠正）。
3. 跑 `ToolGroupExample` 理解"元工具"：`reset_equipped_tools` 让 agent 自己决定激活哪些工具组——**把工具管理本身也工具化**，这是控制上下文膨胀的实战技巧。

### 🧠 深挖自测
- [ ] 读 `tool/ToolSchemaGenerator.java`：`generateParameterSchema()` 怎么从 `Method` 生成 schema？嵌套对象怎么递归？
- [ ] 读 `tool/ToolExecutor.java`：`sortInterceptors()` 和 `applyInterceptorChain()` 如何配合？

### 🎤 面试考点
- "工具数量太多导致 prompt 爆炸怎么办？"（工具分组/元工具激活、`ToolSearchStrategy`、deferLoading）
- "工具执行超时/重试在框架哪一层做的？为什么不在 agent 循环里做？"

---

## 阶段 4：模型抽象与 Provider 生态（4 天）

### 📖 概念
- **策略模式四件套**：`Model` 接口（Strategy）→ `ChatModelBase`（Template Method）→ `ModelProvider`（SPI）→ `ModelRegistry`（Registry + ServiceLoader）
- **双策略维度**：Model 管调用，`Formatter<TReq,TResp,TParams>` 管消息格式转换（每个厂商一份）
- **模型 ID 路由**：`"openai:gpt-4o"`、`"dashscope:qwen-plus"` 前缀 → provider

### 🔍 核心讲解
这是**全项目最标准、最值得抄走复用**的一组设计，画出来是这样的：

```
Model.java            interface  → Flux<ChatResponse> stream(msgs, toolSchemas, options)
   ↑
ChatModelBase.java    abstract   → stream() 是 final（包装 tracing）→ 委托抽象 doStream()
   ↑                                ↑ Template Method：固定流程，开放扩展点
OllamaChatModel / AnthropicChatModel / DashScopeChatModel ...
   
ModelProvider.java    SPI 接口    → providerId() / supports(modelId) / create(modelId)
   └─ META-INF/services/io.agentscope.core.model.spi.ModelProvider
        └─ ModelRegistry 用 java.util.ServiceLoader 自动发现

Formatter.java        interface  → format(Msgs→厂商DTO) / parseResponse(厂商DTO→ChatResponse)
   └─ DashScopeChatFormatter / AnthropicChatFormatter ...  ← 第二个策略维度
```

**为什么 Formatter 独立于 Model？** 消息格式转换（AgentScope Msg ↔ 厂商 DTO）和 API 调用（HTTP/streaming）是两个独立变化轴。有的厂商 API 兼容（Kimi/GLM/DeepSeek 都走 OpenAI 兼容层 `openai/compat/`）但格式细节有差异——**正交分解让兼容层复用调用代码只换 Formatter**。

**彩蛋——SPI 在 fat jar 里的生存**：`agentscope-distribution/agentscope-all` 打 uber-jar 时，shade 插件必须配 `ServicesResourceTransformer` 合并各模块的 `META-INF/services` 文件，否则 ServiceLoader 在 fat jar 里会发现不了任何 provider。**自己项目打 fat jar + SPI 踩坑时，答案在这里**。

### 💻 Demo 实操
1. 读 `model/ModelRegistryExample.java`（examples 里现成的），跑通，观察 `resolve()` 的解析顺序：具名注册 → 缓存 → 用户工厂 → SPI。
2. **对比阅读**（这是本阶段核心作业）：
   - `agentscope-extensions-model-ollama/.../OllamaModelProvider.java`（最简，~几十行）
   - `agentscope-extensions-model-anthropic/.../AnthropicChatModel.java` + 整个 `formatter/` 包（最全）
   对照填写：「实现一个新厂商需要写哪几个类？改几处核心代码？」（答案：3 个类，0 处核心改动）
3. 思考题：`ReActAgent.modelForCall()` 里的 `SessionModels.ManagedModel` 是什么模式？（Decorator——首信号 failover 到 fallbackModel）

### 🎤 面试考点
- "设计一个支持多模型厂商的 SDK，怎么做扩展性？"（把上面那张图讲出来就是满分答案）
- "Java SPI 和 Spring 的依赖注入各适合什么场景？SPI 有什么坑？"（fat jar 合并、无序加载、异常吞噬）

---

## 阶段 5：ReActAgent 主循环 —— 本项目的心脏（1.5 周）

### 📖 概念
- **Builder 模式 + 防御性拷贝**：`List.copyOf(middlewares)`、`Toolkit.copy()`——构造完成后不可变
- **Template Method**：`AgentBase.runLifecycle()` 固定生命周期，子类实现 `doCall()`
- **递归 Reactor 组合替代 while 循环**
- **Reactor Context 作为 scoped DI**（替代 ThreadLocal）

### 🔍 核心讲解
`ReActAgent.java` 约 6800 行，**严禁顺序阅读**。按这个顺序剥（行号基于当前版本，可能漂移）：

**第 1 层：Builder（~5880 行起）** — 注意 build 时的防御性拷贝：middleware 列表、toolkit 都被快照，之后外部修改不影响 agent。**为什么重要**：agent 是长生命周期对象，builder 是短生命周期配置——不拷贝就是埋定时炸弹。

**第 2 层：`buildAgentStream()`（~1330 行）— 全框架的枢纽**：
```
call(msgs)         ─┐
                    ├→ buildAgentStream() ─→ Flux.create(sink) ─→ [AgentStartEvent ... AgentEndEvent]
streamEvents(msgs) ─┘         ↑ call() 只多做了: filter(AgentResultEvent).takeLast(1).next()
```
三个设计决策值得背下来：
1. **单一管道**：阻塞和流式共享同一核心 → 终态天然一致，不会有"流式有事件、阻塞没有"的分裂
2. **sink 注入 Reactor Context**（`EVENT_SINK_KEY`）：深层代码不发回调，而是从 Context 拿 sink 发事件——**响应式代码替代 ThreadLocal 的标准手法**
3. **整条流被 ON_AGENT 中间件洋葱包裹**：连"agent 的执行本身"都可被拦截

**第 3 层：`CallExecution` 内部类（~2100 行起）— 循环本体**：
```
doCallInner(msgs)          入口分流：待确认工具结果？恢复 vs 新开始
  → executeIteration(iter)
      → reasoning(iter)     pre-reasoning hook → 构建工具 schema → ON_REASONING 洋葱 → 累积流式块
      → runPostReasoningPipeline()   决策：结束？停止请求？无工具调用（循环回 reasoning）？→ acting
      → acting(iter)        提取 ToolUseBlock → 权限门 → ON_ACTING 洋葱 → runToolBatch → post hook
      → executeIteration(iter + 1)   ← 递归，循环的"回边"
  → summarizing()          iter >= maxIters 时的终结总结
```
**为什么递归而不用 while？** 每一轮都是一条独立的 Reactor 链，取消信号（`dispose()`）、背压、错误传播沿链自然流动；while 循环里这些都要手工管理。这是**响应式思维和命令式思维的分水岭**——面试聊"你在 Reactor 踩过什么坑"时的高级素材。

**第 4 层：并发控制**：`callSerializationKey`（~853 行）按 `(userId, sessionId)` 串行化同一会话的并发调用——多线程同时 `call()` 同一 session 不会交叉写状态。**生产系统防数据竞争的现实答案**。

### 💻 Demo 实操
1. **事件时序追踪实验**：给阶段 2 的审计器加上"迭代编号"，确认一次带 2 个工具的任务产生了几轮 MODEL_CALL，画出完整时序图（这就是你的面试白板图）。
2. **maxIters 实验**：builder 里设置很小的 maxIters，观察 `summarizing()` 行为。
3. **取消实验**：`streamEvents()` 返回的 Flux `take(Duration)` 后取消，观察事件流在哪一层中断、状态是否一致。

### 🧠 深挖自测
- [ ] 结构化输出强制循环为什么可以 `ignoreMaxIters=true`？（~3081 行：schema 校验失败时重试不算用户的迭代预算）
- [ ] `ReasoningContext`（accumulator 包）为什么存在？（流式块累积成完整 Msg 的缓冲器）

### 🎤 面试考点
- "你的 agent 框架里阻塞 API 和流式 API 怎么保证行为一致？"
- "响应式代码里怎么传递请求级上下文（traceId、user）？为什么不能用 ThreadLocal？"

---

## 阶段 6：Middleware 扩展体系（3 天）

### 📖 概念
- **洋葱模型**：`MiddlewareChain.build()` 把中间件列表**从后往前折叠**，最后一个包住核心逻辑，第一个在最外层
- **三种扩展形态**：洋葱钩子（before/after + next）、顺序 pipeline（`onSystemPrompt`：`Mono<String>` 链式变换）、纯通知（`onAgentStateReady`）
- **六个扩展点**：`onAgent` / `onReasoning` / `onActing` / `onModelCall` / `onSystemPrompt` / `onAgentStateReady`

### 🔍 核心讲解
**1. 折叠方向**（`MiddlewareChain.build()`）：`foldRight`——列表 `[A, B, C]` + 核心 `core` 变成 `A(B(C(core)))`。A 声明 `order()` 最大 → 在最外层。**与 Spring Interceptor / OkHttp Interceptor / Koa middleware 同一血统**，一次学透四家。

**2. 性能设计**：`ReActAgent.groupMiddlewares()`（~1061 行）在**构造时**一次性把中间件按扩展点分组进不可变 `EnumMap`，运行时零排序开销——"把每次调用的开销移到构造期"的通用优化思路。

**3. 签名设计**（已验证，注意 `activePoints()` 显式声明参与点——不参与的方法完全不进链，零开销）：
```java
class MyMiddleware implements MiddlewareBase {
    @Override
    public Set<ExtensionPoint> activePoints() {
        return EnumSet.of(ExtensionPoint.ON_REASONING);   // 只挂 reasoning 钩子
    }
    @Override
    public int order() { return 100; }                    // 大者在外层
    @Override
    public Flux<AgentEvent> onReasoning(Agent agent, RuntimeContext ctx,
            ReasoningInput input, Function<ReasoningInput, Flux<AgentEvent>> next) {
        // before 逻辑
        return next.apply(input).doOnComplete(/* after 逻辑 */);
    }
}
```

### 💻 Demo 实操
自己写三个中间件（API 已验证）：
1. **TimingMiddleware**：`onReasoning` 记录每轮模型调用耗时，`onAgent` 汇总输出
2. **SensitiveWordFilterMiddleware**：`onSystemPrompt` 往系统提示词追加合规要求（它是 `Mono<String>` 变换链，不是洋葱）
3. **ToolAuditMiddleware**：`onActing` 拦截所有工具调用记审计日志，`doOnNext` 里监听 `ToolProgressEvent`

跑法：builder 里 `.middleware(new TimingMiddleware()).middleware(...)` 叠加，故意给两个中间件不同 `order()` 观察执行顺序。

### 🎤 面试考点
- "给你们的 agent 系统加一个全链路灰度开关，加在哪一层？为什么？"（middleware 洋葱最外层 / onAgent）
- "责任链和装饰器在 middleware 这里是同一个东西吗？"（是——洋葱链每层既是处理节点又是包装器；区别只在语义视角）

---

## 阶段 7：状态与会话持久化（5 天）

### 📖 概念
- **AgentState**：会话上下文（消息历史）+ 权限上下文 + 工具上下文，2.0 起取代独立 Memory
- **AgentStateStore SPI**：按 `(userId, sessionId, key)` 寻址的 Repository 模式
- **乐观并发**：`getVersioned` / `saveIfVersion` + `ConflictPolicy`（OVERWRITE/FAIL/APPEND_MERGE）
- **Session Log**：journal + lease + checkpoint 的事件溯源层

### 🔍 核心讲解
**1. 接口默认方法做能力降级**（`state/AgentStateStore.java`）：`saveIfVersion()` 有 default 实现，降级为无条件 `save()`；`supportsVersioning()` 让调用方探测——**SPI 允许实现方"部分实现"而不破坏契约**。比抛 `UnsupportedOperationException` 优雅得多。

**2. CAS 冲突处理**（`ReActAgent.persistAgentStateCas()`）：多实例部署时两个 pod 同时写同一 session → 版本冲突 → 三种策略。**分布式面试题的现实版**。

**3. 事件溯源（session/ 包）**：`SessionRecorder`（逐事件按序落盘 + checkpoint）、`SessionLog`（journal SPI：head/lease/scan/append）、`SessionRecovery`。和 Kafka/数据库 WAL 同构，但代码量小到一周能读完——**读源码学分布式，这是性价比最高的一课**。

### 💻 Demo 实操
1. **持久化会话**（API 已验证）：
```java
AgentStateStore store = new JsonFileAgentStateStore(
        Paths.get(System.getProperty("user.home"), ".agentscope", "my-sessions"));
ReActAgent agent = ReActAgent.builder()
        .name("Assistant")
        .sysPrompt("...")
        .toolkit(new Toolkit())
        .stateStore(store)              // ← 配了就自动 load/save
        .defaultSessionId("alice")
        .model("dashscope:qwen-plus")
        .build();
// 第一次跑："我叫 Alice" → 退出 → 第二次跑问"我叫什么"（同一 sessionId）
System.out.println(agent.getAgentState().getContext().size());  // 恢复的消息数
```
2. **写一个自定义 Store**：实现 `AgentStateStore`，落库到你自己的 SQLite/MySQL（参照 `agentscope-extensions-jdbc` 的实现，别抄，先自己写再对照）。
3. 打开 `~/.agentscope/.../sessions/` 下的 JSON 文件，人工读一遍 `AgentState` 的结构——面试画状态结构图就靠这个。

### 🎤 面试考点
- "agent 服务多实例部署，同一会话并发写入怎么处理？"（会话级串行化 + 版本 CAS + 冲突策略，把阶段 5 的 `callSerializationKey` 和这里的 `saveIfVersion` 串成一条答案）
- "会话历史无限增长怎么办？"（harness 的 `CompactionMiddleware` / `ConversationCompactor`——预告阶段 9）

---

## 阶段 8：权限系统与 HITL（3 天）

### 📖 概念
- **规则管道**：deny 规则 → ask 规则 → 工具自检（bypass 免疫）→ allow 规则 → 模式默认（EXPLORE/ACCEPT_EDITS/BYPASS/...）
- **deny-overrides-allow**：安全系统铁律——显式拒绝永远优先
- **基于调用的 HITL 往返**：ASK 判决挂起回合，发 `RequireUserConfirmEvent`，**下一次 `agent.call()` 携带 ConfirmResult 恢复**
- **跨回合状态机**：`ToolCallState`（ASKING → ALLOWED/DENIED → ...）

### 🔍 核心讲解
**HITL 的正确建模**（本框架最巧的设计之一）：确认不是回调、不是阻塞线程，而是**把"人类审批"建模成一次额外的 agent 调用**：
```
turn 1: agent.call("删掉这个文件")
   → 权限门: ASK → tool_use 状态 = ASKING → 发 RequireUserConfirmEvent → 本回合挂起
turn 2: agent.call(confirmResults)   ← 人类的决定作为下一次调用进入
   → applyConfirmResults → 恢复 ASKING 的 tool_use → 继续循环
```
对比常见错误设计（弹窗阻塞线程 / 全局 Future 等待）：调用式往返**天然支持服务重启后恢复**——审批状态在 `AgentState` 里持久化了，pod 重启不丢。**这是"面试聊人机协同"时的降维打击素材**。

**双层防护回顾**：`PermissionEngine`（agent 层，策略：该不该做）+ `ToolCallInterceptor`（toolkit 层，执行：参数改写/审计/挂起）。参考对照：这是 **认证 vs 授权、策略执行点 vs 策略信息点**（PDP/PEP）思想的 agent 版。

### 💻 Demo 实操
1. 跑 `hitl/PermissionHITLExample.java` 和 `hitl/InterruptionExample.java`，观察确认往返的完整事件序列。
2. 跑 `tool/PermissionContextExample.java`。
3. 实验：给一个删文件工具配 deny 规则，观察 BYPASS 模式下它是否仍被拒绝（bypass 免疫的设计意图）。

### 🎤 面试考点
- "你们的 agent 能自主执行 shell 命令，安全怎么保障？"（规则管道 + 双层防护 + 沙箱（阶段 9）+ 审计，四层讲全）
- "LLM 系统的'人工确认'怎么设计才不会把服务搞挂？"（调用式往返 + 状态持久化恢复）

---

## 阶段 9：Harness 高层架构（1.5 周）

### 📖 概念
- **组合优于继承**：`HarnessAgent` 持有 `private final ReActAgent delegate`，**从不继承/fork**
- **一切皆 Middleware**：子代理、沙箱、压缩、团队、收件箱全部以 `MiddlewareBase` 实现注入
- **文件系统策略家族**：`Local` / `Remote` / `Sandbox` / `Overlay` / `Composite` 可组合
- **Channel SPI**：钉钉/飞书/企微/GitHub 各自实现 `Channel` 接口接入同一个 Gateway

### 🔍 核心讲解
**1. 为什么"一切皆 Middleware"能成立？** 打开 `HarnessAgent.Builder`（~2864-3312 行）：子代理（`SubagentsMiddleware`）、上下文压缩（`CompactionMiddleware`）、沙箱生命周期（`SandboxLifecycleMiddleware`）、收件箱（`InboxMiddleware`）、团队（`TeamsMiddleware`）……全是对 `ReActAgent.Builder` 的 `inner.middleware(...)` 注入。**能力 = 有序中间件 + 若干工具**，这个等式让 core 永远不需要为上层能力改代码。对比市面上"继承 ReActAgent 覆写方法"的做法：**继承把变化耦合进类层次，组合把变化表达为可插拔序列**。这是面试聊 OCP（开闭原则）的最佳实战案例。

**2. 文件系统的组合模式**：`OverlayFilesystem`（读远端写本地）、`CompositeFilesystem`、`RoutedSandboxFilesystem`——agent 的"文件系统"是一个**可组合的策略对象**，而不是一个路径。沙箱化 coding agent 的地基。

**3. Channel 接入模型**：`Channel` SPI（init → start → stop）→ `ChannelRouter.resolveRoute()` 找到目标 agent/session → `Gateway.run()`。加一个 IM 平台 = 实现一个接口，零核心改动（和阶段 4 的 Provider 同构——**好的架构里每种扩展点长一个样**）。

### 💻 Demo 实操
examples 里 `documentation2/harness/` 有完整的成组示例，按此顺序跑并读源码：
1. `workspace/WorkspaceSetupExample` → `WorkspaceLocalShellExample` → `WorkspaceSandboxExample`
2. `subagent/SubagentStreamingExample`
3. `memory/MemoryCompactionExample`（上下文压缩——阶段 7 遗留问题的答案）
4. `channel/ChannelSendExample` → `GatewayMultiAgentExample`
5. `planmode/PlanModeAutoExample`

### 🎤 面试考点
- "用继承和用组合/中间件扩展 agent 框架，各自的代价？"（继承：类爆炸、core 侵入、能力不可拆；组合：顺序管理、调试链路长）
- "多 agent 编排你了解哪些模式？"（子代理委派 / 团队 TeamClient / A2A 跨进程协议——注意讲清**进程内 vs 跨进程**的分界）

---

## 阶段 10：生产化工程（1 周）

### 📖 概念
- SSE 流式 Web 服务、会话恢复、HITL 前后端协议
- 响应式测试：Reactor `StepVerifier`
- 可观测：OTel（`OtelTracingMiddleware`）、token 成本核算（阶段 2 审计器的生产版）
- 平台分层：gateway / controlplane / dataplane / scheduler

### 🔍 核心讲解
**1. 响应式怎么测**：`agentscope-core/src/test/` 是活教材——Mockito mock `Model` 返回构造好的 `Flux<ChatResponse>`，`StepVerifier` 断言事件序列。**不用真 API key、不打真模型**，测试快且稳。这是 agent 工程和普通 CRUD 工程测试观差异最大的地方。

**2. 平台分层**（`agentscope-service/README.md`）：
```
service-gateway     统一入口（Spring Cloud Gateway）
service-controlplane Go 管理面：端点/调用/发布 + React UI
service-dataplane   数据面：HarnessAgent 运行时、SSE、HITL、工作队列   ← agent 真正跑的地方
service-scheduler   调度面：hands、IM 适配、外发投递
```
管理面用 Go、数据面用 Java 的混语言架构——按团队技能和生态选语言，而不是宗教。

### 💻 Demo 实操
1. 跑 `streaming/StreamingWebExample.java`（Spring Boot + SSE），curl 打 SSE 端点观察事件流。
2. **StepVerifier 练习**：给阶段 6 的 TimingMiddleware 写单测——mock 一条两个事件 的 Flux，断言中间件透传且日志被调用。
3. 跑 `agentscope-chat` 完整应用（无需 API key），重点看它的 SSE 断线重连和 checkpoint 续传怎么用事件流实现。

### 🎤 面试考点
- "agent 服务怎么做自动化测试？"（mock Model 层 + StepVerifier 事件序列断言 + 集成层用离线 demo model）
- "生产 agent 的可观测性要覆盖哪些维度？"（traceId 贯穿 / token 成本 / 工具成功率和耗时 / 事件回放）

---

## 毕业项目：Mini Coding Agent（2 周）

**目标**：不依赖 harness 的 workspace/sandbox，自己用 core + middleware 组一个"能干活的 agent"，作为面试的演示项目。

### 需求清单（每条对应一个已学阶段）
1. `@Tool` 实现文件读写 + shell 执行工具，shell 工具挂 **PermissionEngine deny/ask 规则**（阶段 3、8）
2. **TimingMiddleware** 记录每轮耗时 + token 审计（阶段 6、2）
3. **CompactionMiddleware 思路自实现**：上下文超 N 条消息时自动摘要压缩（阶段 7、9）
4. **AgentStateStore** 持久化会话，支持重启恢复（阶段 7）
5. **Spring Boot + SSE** 暴露流式接口，前端打字机渲染 + HITL 确认按钮（阶段 10、8）
6. （加分）自定义 `ModelProvider` 接一个本地 Ollama（阶段 4）

### 验收标准
- 一条命令启动，全程不重启完成：读文件 → 请求改文件 → 前端弹确认 → 执行 → 汇报
- 项目 README 里有**你自己画的三张图**：事件时序图、middleware 洋葱图、状态流转图
- 核心链路有 StepVerifier 单测

---

## 面试冲刺：高频题清单（按阶段组织，模拟面试用）

**原理层**
1. ReAct / Plan-and-Execute / Reflexion 的区别与选型？
2. function calling 的完整协议流程？工具参数 schema 怎么生成、strict 模式是什么？
3. 多轮对话的上下文管理策略有哪些？（截断/摘要压缩/外置记忆检索，各自的坑）
4. 结构化输出怎么实现？模型不支持原生 structured output 时怎么办？（框架的 fallback：工具调用强制 + 校验重试循环）

**架构层**
5. 你的 agent 框架里阻塞 API 和流式 API 怎么保持一致？（单管道 + filter 末端事件）
6. middleware 链怎么实现？和 Spring Interceptor 区别？
7. 多厂商模型接入的扩展性设计？（SPI + Registry + 双策略轴，讲出 fat jar 合并坑直接加分）
8. 会话持久化的并发冲突怎么处理？
9. HITL 审批怎么设计才能在服务重启后恢复？
10. 子代理和 A2A 协议的适用边界？

**工程层**
11. agent 的自动化测试策略？怎么 mock 模型层？
12. token 成本怎么核算和控制？
13. agent 执行 shell/写文件的权限模型？沙箱方案？
14. 流式断线重连 + 事件回放怎么做？（replyId + 事件 ID + 事件存储）

**Java 语言层（结合本框架源码答，直接体现深度）**
15. sealed interface + record 的实际应用场景？（ContentBlock 层级）
16. Builder 模式里防御性拷贝的必要性？
17. 响应式编程里 ThreadLocal 为什么失效？替代方案？（Reactor Context / ScopedValue）
18. Java SPI 机制和坑？

**答题万能框架**：结论先行 → "在我读过的 agentscope-java 实现里是这么做的…"（具体类名/机制）→ 权衡与边界 → 如果是我设计会怎么改。**能说出具体类名的那一刻，你就和 90% 的候选人区分开了。**

---

## 毕业检查清单

- [ ] 能白板画出 ReAct 循环完整时序图（含事件流 + 迭代编号）
- [ ] 能白板画出 middleware 洋葱图并解释折叠方向和 order 语义
- [ ] 能白板画出 Model/Provider/Formatter/Registry 四件套类图
- [ ] 能解释 call() 和 streamEvents() 为什么天然一致（指出具体方法）
- [ ] 能解释 HITL 调用式往返和它对服务重启的鲁棒性
- [ ] 能解释双洋葱（agent 层权限 + toolkit 层拦截器）的分层理由
- [ ] 写过：自定义工具 + 自定义 middleware + 自定义 StateStore + StepVerifier 单测
- [ ] 读过 ReActAgent 的 buildAgentStream → CallExecution 主链路
- [ ] 毕业项目能现场演示，README 三张图齐全
- [ ] 上面 18 道面试题每道能脱稿讲 3 分钟

---

## 附录 A：模式速查表

| 模式 | 位置 | 一句话 |
|---|---|---|
| Builder + 防御性拷贝 | `ReActAgent.Builder` | 构造后不可变，外部改动不泄漏 |
| Template Method | `AgentBase.runLifecycle` / `ChatModelBase.stream()` | final 固流程，开放扩展点 |
| Strategy | `Model` / `Formatter` / `ToolSearchStrategy` | 每厂商一份，可替换 |
| 洋葱/责任链 | `MiddlewareChain.build()` / `ToolExecutor` | foldRight 折叠，order 定内外 |
| Adapter | `ReflectiveFunctionTool` / `A2aAgent` | Method→AgentTool，远程→本地 |
| Registry + SPI | `ModelRegistry` + `ModelProvider` | ServiceLoader 自动发现厂商 |
| Null Object | `NoOpToolEmitter.INSTANCE` | 可选依赖不判空 |
| Observer | `ToolEmitter` / `ToolExecutionCallback` | 进度推送不进 LLM 上下文 |
| Decorator | `SessionModels.ManagedModel` | 模型 failover 包装 |
| 组合优于继承 | `HarnessAgent`(delegate) | 能力=中间件+工具，不 fork core |
| Sealed + record | `ContentBlock` | 编译期穷尽性，不可变值对象 |
| Reactor Context | 事件 sink / RuntimeContext | 响应式 scoped DI |
| 乐观并发 CAS | `AgentStateStore.saveIfVersion` | 多实例写同一会话 |
| 事件溯源 | `session/SessionRecorder` | journal+checkpoint+recovery |

## 附录 B：源码阅读地图（agentscope-core 包 → 阶段对应）

```
message/   → 阶段2   model/     → 阶段4   tool/      → 阶段3
middleware/→ 阶段6   formatter/ → 阶段4   agent/     → 阶段5(含 ReActAgent.java)
state/     → 阶段7   session/   → 阶段7   permission/→ 阶段8
skill/     → 阶段9*  workspace/ → 阶段9*  (*核心只有 SPI，实现在 harness)
```

**阅读纪律**：每个包先读接口/抽象类 → 再读一个最简实现 → 最后读一个最全实现；同时开着对应 `src/test/`，测试就是行为规格书。
