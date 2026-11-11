# 学习计划（速胜版）

> 本文件只管"做什么、做到什么程度算过"。理论细节去 `PATH.md` 按需查，不通读。
> 节奏：每天 1h（概念 burst ≤30min + 动手 ≥25min）；每个里程碑 4~7 天；全程约 5~6 周。

## 里程碑总览

```
M1 会跑会调(5天) → M2 懂扩展(5天) → M3 读透心脏(7天) → M4 会存续(6天) → M5 架构师(7天) → M6 贡献者(3天)
   PATH 1-3         PATH 4+6          PATH 5             PATH 7+8         PATH 9+10+毕设    上游PR
```

> 跳跃规则：允许提前偷看后面里程碑的内容（比如 M1 时翻 M3 的时序图），
> 但**出口标准不可跳、不可降**；提前学的部分，轮到该里程碑时只需补练习。

---

## M1 会跑会调 —— "我能一周内写出带工具的 agent"（对应 PATH 阶段 1-3）

- [ ] 跑通三个官方示例：BasicChat / ToolCalling / AgentEventStream
- [ ] ex-01 事件审计器：订阅 streamEvents，统计迭代次数、token、工具成功率
- [ ] ex-02 自定义工具：带 ToolEmitter 进度 + 故意抛错观察模型自愈

**出口标准**（全部满足才进 M2）：
- □ 不看笔记，白板画出"用户提问 → 工具调用 → 回答"完整事件时序
- □ 能说清 call() 与 streamEvents() 的关系
- □ ex-01 / ex-02 通过导师 review

## M2 懂扩展 —— "框架的口子我都能接"（对应 PATH 阶段 4+6）

- [ ] 对比读 Ollama vs Anthropic Provider → 写 notes/m2-provider.md
- [ ] ex-03 TimingMiddleware：每轮耗时 + token 记账
- [ ] ex-04 敏感词 SystemPrompt 中间件（pipeline 型，非洋葱）

**出口标准**：
- □ 能画 Provider/Registry/Formatter 四件套类图
- □ 能解释 middleware 折叠方向与 order 语义
- □ 3 分钟脱稿讲"多厂商 SDK 扩展性设计"面试题

## M3 读透心脏 —— "ReActAgent 我读穿过"（对应 PATH 阶段 5，最硬一关，允许拆两次）

- [ ] 源码追踪主线：buildAgentStream → CallExecution → reasoning/acting 递归
- [ ] ex-05 两个实验：流取消中途断掉 + maxIters 强制总结
- [ ] 写 notes/m3-react-loop.md（必须自己画时序图）

**出口标准**：
- □ 能讲"为什么循环是递归 Reactor 而不是 while"
- □ 能讲 Reactor Context 为什么替代 ThreadLocal
- □ 事件时序图能默画

## M4 会存续 —— "重启不丢记忆、危险操作会请示"（对应 PATH 阶段 7+8）

- [ ] ex-06 自定义 StateStore（SQLite，先自己写再对照 jdbc 扩展）
- [ ] 跑 PermissionHITLExample，追踪 ASK → 挂起 → 恢复全程
- [ ] 写 notes/m4-hitl.md

**出口标准**：
- □ 演示同 sessionId 两次运行记忆延续
- □ 能讲 HITL 调用式往返 + 服务重启恢复原理
- □ 能讲多实例写同一会话的冲突处理（CAS + ConflictPolicy）

## M5 架构师 —— 毕业项目 Mini Coding Agent（对应 PATH 阶段 9+10）

- [ ] 文件读写 + shell 工具，shell 挂 deny/ask 权限规则
- [ ] 接入 ex-03 的审计中间件 + 会话持久化
- [ ] Spring Boot SSE 流式 + HITL 确认
- [ ] 整合进 exercises/capstone-mini-coding-agent/

**出口标准**：
- □ 一条命令启动，走通：读文件 → 请求修改 → 确认 → 执行 → 汇报
- □ README 三张图：事件时序 / 洋葱 / 状态流转
- □ 核心链路有 StepVerifier 单测

## M6 贡献者 —— "我在上游有 PR"

- [ ] 从 upstream 挑一个 good first issue / 文档改进
- [ ] 按 CONTRIBUTING.md 流程提 PR 到 agentscope-ai/agentscope-java

**出口标准**：PR 已提交（合并与否不影响达标）

---

## 反馈机制

- **每个练习**：导师 review，给 1 个优点 + 1 个改进点（不多不少，速胜派要的是快反馈）
- **每个里程碑**：PROGRESS.md 写 5 行复盘——最有用的 / 最没用的 / 下个想跳哪
- **任何时刻觉得枯燥或太慢**：直接说，导师当场调整（砍理论、上实操、换主题）
