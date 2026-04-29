# 第四幕 · 可靠与可观测的两张脸

> 这章要解决什么问题？
> Agent 跑完之后, 输出怎么到用户? 为什么 Delivery 和 Events 是**两张完全不同的脸**?

---

## 一

凌晨五点四十五分. 他追到了 `deliverAgentCommandResult` 这个函数.

一个 333 行的文件, 做的事听着简单——"把 agent 的回复送到用户那边".

但他翻着翻着就发现——**有两套输出 surface**.

- 一套叫 **Delivery**: 可靠的, transactional 的, 要 retry 的, 要保证送达的.
- 一套叫 **Events**: fire-and-forget, best-effort 的, 丢了就丢了.

两套 surface 同时在跑, 互不干涉.

---

## 二

他画了一张图:

```
        ┌── agent loop 产出 ──┐
        │                      │
        │   RunResult:         │
        │   { payloads,        │
        │     meta,            │
        │     sessionState }   │
        └──────────┬───────────┘
                   │
         ┌─────────┴─────────┐
         │                    │
         ▼ 可靠                ▼ 旁路
  ╔══════════════╗      ╔════════════╗
  ║   Delivery   ║      ║   Events   ║
  ║              ║      ║            ║
  ║ · 格式化      ║      ║ · emit 即可 ║
  ║ · channel     ║      ║ · 丢了就丢 ║
  ║   transform  ║      ║ · 非阻塞   ║
  ║ · retry      ║      ║            ║
  ║ · bestEffort ║      ║            ║
  ╚═══════╤══════╝      ╚════════╤═══╝
          │                      │
          ▼                      ▼
    用户 (channel)          UI / Gateway
    Telegram/Discord        Log / Monitor
    /CLI stdout             (observability)
```

两条路. 两种 SLA.

- 用户的消息是**可靠层**——必须送到, 不到就 retry, retry 三次还不到就 fallback 到 best-effort, 再不到就报告给 ops.
- 观测事件是**best-effort**——eventbus.emit(), 丢几个无所谓, 丢一片也无所谓.

---

## 三

他在笔记上写下一段话:

> **为什么不把观测事件也做成可靠的?**
>
> 因为**可靠性有代价**——要 retry, 要 dedupe, 要 durable queue, 要 ack 机制. 观测事件每秒产生上千个, **如果每个都可靠, 系统被 observability 压死**.
>
> **可靠性分层**的本质是**承认不同通道的 SLA 不同**.
>
> 用户回复 SLA: 5 秒送达, 99.9% 成功率
> 观测事件 SLA: 尽力, 90% 成功率就行
>
> **把两者混在一个 channel, 你要么牺牲观测性(都做轻), 要么牺牲用户体验(都做重)**.

这让他想起编排国(Claude Architect 笔记)的那课——**Hooks vs Prompt 强制**. 本质是同一个思想:**不同 SLA 的事, 用不同机制做**.

---

## 四

他继续看 `deliverAgentCommandResult` 内部. 发现一个**优雅的分支结构**:

```ts
if (opts.json) {
  // JSON envelope 模式 (给 gateway / CI)
} else if (deliver === true) {
  // 真送到 channel
} else {
  // CLI 模式: print 到 stdout
}
```

**三种模式, 三个消费者**.

- `--json` 给 **gateway** (要 structured envelope 路由)
- `deliver=true` 给 **channel** (Telegram/Discord, 真实发)
- 默认给 **CLI** (stdout, 开发者看)

"这也是'同一件事三个 SLA'的另一种形态." 他在笔记里说. "不是'有两个人用', 是'有三种使用场景, 每种要求不同'."

---

## 五

他突然意识到今晚 bug 的一种可能性——

"如果 agent 跑完, **Delivery 失败了(比如 Telegram 的 token 临时过期), 但 agent lifecycle.end 的 event 先 emit 了**——那监控会显示'正常结束'; 但**用户没收到**."

他翻到 `agent-command.ts:904-929`:

```ts
onAgentEvent: (evt) => {
  if (
    evt.stream === "lifecycle" &&
    (evt.data.phase === "end" || evt.data.phase === "error")
  ) {
    lifecycleEnded = true;
  }
}
// ...
if (!lifecycleEnded) {
  emitAgentEvent({
    runId, stream: "lifecycle",
    data: { phase: "end", startedAt, ... }
  });
}
```

这是**兜底 emit**. 确保 lifecycle.end **一定被触发**, 哪怕 agent 自己忘了.

"等等——" 他盯着代码, "所以**lifecycle.end 的 emission 是 best-effort, 和 delivery success 脱钩的**."

"这意味着我今晚看到的 lifecycle.end **覆盖率 25%**——如果 agent 正常结束也会 emit, 但 delivery 如果失败了, lifecycle.end **依然 emit**."

"**25% 的 lifecycle.end 意味着 agent 在跑 75% 的时候直接死了**, 不是 delivery 失败."

——但 他 没 把 这 个 推论 当 结论. 他 又 加 了 一 步.

"——**反向 验证**." 他 打开 events 表, 过滤 那 75% 没 emit 过 lifecycle.end 的 runId, 看 它们 的 error stream:

```sql
SELECT runId, evt.data.message
FROM agent_events
WHERE stream = 'error'
  AND runId IN (没有 lifecycle.end 的 runId 集合)
LIMIT 20;
```

仪表盘 跳 出 来——

```
[75 / 75]  unexpected exit (no stopReason)
[75 / 75]  resolveModelAsync failed: provider=... not in registry
[75 / 75]  ensureRuntimePluginsLoaded threw: ...
```

——**75 条 里, 75 条 都 有 'unexpected exit' 或 类似 异常 标记**. 没有 一条 是"成功 跑完 但 兜底 emit 没 触发". 验证 通过.

他 在 笔记 边上 写:

> **推论 不 等于 结论. lifecycle.end 缺失 + 异常 标记 同时 出现, 才能 把 bug 定位 到 agent 内部.**

这个推理让他重新定位 bug 的位置——**在 agent 内部, 不在 delivery 层**.

---

## 六

他关掉 delivery.ts, 打开 agent-events 的仪表盘.

"看 `error` stream 里最近 30 分钟的事件."

仪表盘跳出来几行——

```
[error] runId=abc123 agent embedded run unexpected exit (no stopReason)
[error] runId=def456 resolveModelAsync failed: provider=chutes not in registry
[error] runId=ghi789 ensureRuntimePluginsLoaded threw: manifest.schema.v2 invalid
```

**第三条**让他坐直了.

"**manifest.schema.v2 invalid**."

这是第三幕(manifest-first)埋下的伏笔——**manifest 坏了, runtime 加载失败**. 但 manifest 坏了应该早期发现, 为什么只在 agent run 时才炸?

他需要去看最近一次的 plugin 更新——很可能刚才的一次 deploy 推了个**有 bug 的 manifest**.

凌晨六点. 有进展了.

---

## 关键概念

- **双输出 surface**: Delivery (可靠) + Events (best-effort). 两种 SLA 分开.
- **三种 delivery 模式**: JSON / channel / stdout — 对应 gateway / 真用户 / 开发者.
- **兜底 emit**: lifecycle.end 由外层兜底, 避免 agent 内部异常导致事件丢.
- **事件 ≠ 交付**: lifecycle.end 触发 ≠ 用户收到回复. 监控要分开看.

## 我的理解

- **可靠性和可观测性应该分通道**, 不要混成一条.
- **监控 lifecycle.end 覆盖率**比监控 error rate 更早发现"静默失败".
- **兜底 emit 是一种工程美德**——不相信自己的代码路径总是跑完.

## 类比 · 邮局和广播电台

```
╔═══ 邮局 (Delivery) ═══╗     ╔═══ 广播站 (Events) ═══╗
║                       ║     ║                        ║
║  · 投递单             ║     ║  · 实时发送             ║
║  · 签收                ║     ║  · 丢几个听众听不到也没事║
║  · 未送达要登记         ║     ║  · 不重播                ║
║  · 要 retry           ║     ║  · 追求覆盖广 ≠ 覆盖全   ║
╚═══════════════════════╝     ╚════════════════════════╝
```

一封录取通知书你得保证送到. 天气预报你想播就播. **两种信息, 两种 SLA, 两种机制**.

## 遗留问题

- 兜底 emit 里如果 emit 本身抛错会怎样? 我希望是 swallow, 但不确定.
- 如果有一天产品要求"观测事件也要可靠"(比如计费场景), 能不能**给 events 加一个 durable sink**而不动 delivery? 这是个设计题.

## 与其他章节的连接

- ← [[ch03-manifest|第三幕]]
- → [[ch05-two-doors|第五幕]]: 到此我已经追明白了"agent 如何被调用"的出站链. 但我还没看**入站**——消息怎么从 Telegram 走到 agent 的?**答案让我意外**——不是一条路, 是两条.
