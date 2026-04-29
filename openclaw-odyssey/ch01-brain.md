# 第一幕 · 1111 行的大脑

> 这章要解决什么问题？
> 一个 agent 的 CLI 命令为什么需要 1111 行？这些代码在解决什么问题？

---

## 一

凌晨三点三十分. 他打开了 `src/agents/agent-command.ts`.

屏幕上跳出文件大小——**1111 行**.

他愣住了. 1111 行, 干一件事——让 CLI 能跑 `openclaw agent --message "hello"`. 这不就是**解析参数, 调 agent, 打印结果**吗? 怎么需要 1111 行?

他开始读文件顶部的 imports——

- 30+ 个本地模块 import
- 4 个 `.runtime.ts` 懒加载包装
- Session / Model / Skills / Auth / ACP / Channel / Routing / Logger... 每一个概念一个 import

"这不是解析参数." 他在笔记里写. "**这是一个调度中枢**."

---

## 二

他翻到 line 424——`agentCommandInternal`.

这个函数签名就是他今晚的路标:

```ts
async function agentCommandInternal(
  opts: AgentCommandOpts & { senderIsOwner: boolean },
  runtime: RuntimeEnv = defaultRuntime,
  deps: CliDeps = createDefaultDeps(),
) { ... }
```

**三个参数**: opts(用户意图), runtime(环境), deps(CliDeps 依赖容器).

他认出了——**这是依赖注入**. 如果你想写测试, 可以塞一个 mock runtime 进来. 如果你想换 channel, 可以塞另一个 deps.

他在边上写:

> **参数不是"配置", 是"可替换的接缝"**.

---

## 三

函数体分两条路径.

```ts
if (acpResolution?.kind === "ready" && sessionKey) {
  // 路径 A: ACP (Agent Client Protocol)
  await acpManager.runTurn({...})
} else {
  // 路径 B: 本地 agent loop
  await runWithModelFallback({
    run: () => attemptExecutionRuntime.runAgentAttempt({...})
  })
}
```

他盯着这两条路径看了一会儿.

"路径 A 是给 ACP 接入的外部 agent——比如 Codex, 比如 Claude Code CLI. 路径 B 是本地的 embedded loop."

"**一个 agent command 入口, 支持两种完全不同的 agent runtime**. 这是 OpenClaw 的第一个'产品级'决定——**不绑死 agent 实现**."

如果未来有新的 agent 协议(比如 OpenAI Agent SDK 开放一个接入口), OpenClaw 只要加第三条路径就行. **路径本身可插拔**.

---

## 四

他继续往下看. 路径 B 里是:

```ts
runWithModelFallback({
  run: (providerOverride, modelOverride, runOptions) => {
    return attemptExecutionRuntime.runAgentAttempt({...});
  }
})
```

——一个 **callback 形式的 retry 循环**.

外层负责"如果这个 provider/model 失败了, 试下一个". 内层负责"单次尝试". 内外层职责完全分离.

"这比简单 try/catch 高级多了." 他说. "**try/catch 是错误处理, runWithModelFallback 是降级策略.** 前者反应, 后者设计."

他在笔记里画了一张图:

```
agentCommand                              ← L0  入口
  └── prepareAgentCommandExecution        ← L1  context 收集
        └── runWithModelFallback          ← L2  provider/model 降级
              └── runAgentAttempt         ← L3  单次尝试
                    └── runEmbeddedPiAgent ← L4  agent loop 实体
```

"**5 层**. 入口 + 4 层 嵌套. 每一层解决一个独立问题."

- L0 `agentCommand`: 参数解析 + 公共入口
- L1 `prepare...`: 收集一切 context (session / skills / model / auth)
- L2 `runWithModelFallback`: provider/model 降级
- L3 `runAgentAttempt`: 单次尝试 (CLI vs Embedded 分支)
- L4 `runEmbeddedPiAgent`: agent loop 实体

---

## 五

他写笔记时手不自觉停住.

**这 4 层嵌套, 每一层都是自己曾经省略掉的一件事**.

——他 又 把 鼠标 推 回 文件 顶 端, 慢慢 往 下 滚 看 这 1111 行 的 **真实 分布**.

```
line  1 - 100   imports + types
line 100 - 250  arg parsing
line 250 - 424  公共校验 + dry-run 分支
line 424 - 650  agentCommandInternal 主体
line 650 - 800  session 管理 (建/恢复/超时)
line 800 - 950  delivery + lifecycle.end 兜底
line 950 -1111  错误处理 + 退出码 + 测试钩子
```

他 看 着 这 个 分布, **手指 停 在 滚 轮 上**.

——真正 调 模型 的 那 几 行 在 line 770 附近. 整 个 文件 1111 行, **真正 跟 LLM 通信 的 不到 50 行**.

剩下 的 1060 行, 全部 是 这 五 件 事:

- 他的旧 chatbot 没有"公共入口抽象"——直接写在 webhook handler 里
- 没有 "prepare" 步骤——所有东西塞在一个 function 里
- 没有 fallback 降级——OpenAI 挂了用户就报错
- 没有"单次尝试"概念——一次就是一次
- 没有 lifecycle 兜底——agent 自己崩了就崩了, 监控也不知道

他想起昨天晚上看见的那 75% 失败率. 如果是他的旧系统, 这 75% 里:
- 有 30% 是 OpenAI 临时 503 (**没 fallback**)
- 有 20% 是 rate limit (**没 retry**)
- 有 15% 是 session 丢失 (**没 session 抽象**)
- 剩下 10% 是真 bug

"生产级 agent harness 的价值," 他在笔记最后写, "**不是让 agent 更聪明, 是让这 60% 的 ops 失败消失**."

他抬头看表——凌晨四点十分. 他还没修好 bug, 但他开始**重新理解这个系统存在的理由**.

---

## 关键概念

- **`agentCommandInternal`**：1111 行 agent 命令大脑. Session/Model/Skills/Auth/ACP/Delivery 的集线器.
- **依赖注入 (CliDeps + RuntimeEnv)**: 让每一层都能被替换. 测试友好 + 可插拔.
- **ACP vs 默认路径**: 支持外部 agent 协议接入. 不绑死本地 loop.
- **4 层嵌套**: command → prepare → fallback → attempt → run. 每层独立问题.

## 我的理解

- 1111 行**不是 over-engineering**, 是对"所有可能失败点"的显式处理.
- 真正 production-grade 的 agent command **不是"简单封装", 是"调度层"**.
- **ops 失败 > 模型失败**. 用户看到的"agent 不行"大多数是 ops 问题, 不是模型.

## 类比

像一栋医院大楼的**急诊分诊台**.

- 接诊(agentCommand): 把所有急诊入口归一
- 评估(prepare): 问你哪里疼, 取医保卡, 查过敏史
- 分流(runWithModelFallback): 内科不行转外科, 外科不行转骨科
- 看诊(runAgentAttempt): 单次就诊
- 治疗(runEmbeddedPiAgent): 具体操作

每一层**都是独立的协议**, 中间任何一层都可以换——比如把分流换成 AI 分诊器, 不影响上下游.

## 遗留问题

- ACP 路径什么场景会用? 我现在还没碰到过, 但知道它在.
- `prepareAgentCommandExecution` 里具体收集了什么, 值得再深入看一层.

## 与其他章节的连接

- ← [[ch00-page|序章]]
- → [[ch02-loop-not-here|第二幕]]: 我跟着 `runAgentAttempt` 往里追, 想看看 agent loop 长什么样. **结果发现了一件让我震惊的事**.
