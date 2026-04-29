# 第二幕 · 原来 loop 不在我家

> 这章要解决什么问题？
> 跟着 `runAgentAttempt` 往里追, 他发现**真正的 agent loop 根本不在 OpenClaw 代码里**. 这是什么意思?

---

## 一

凌晨四点半. 他跟着 `runAgentAttempt` 往里钻.

- `runAgentAttempt` → `runEmbeddedPiAgent`
- `runEmbeddedPiAgent` 2074 LOC — 看名字是 agent loop, 里面 for(;;) 一堆 retry 逻辑
- 真正循环体在 `runEmbeddedAttemptWithBackend`
- 再往里是 `runAgentHarnessAttemptWithFallback` → `harness.runAttempt`
- 再往里是 `runEmbeddedAttempt` in `run/attempt.ts` — 2486 LOC
- 然后他看到这一行——

```ts
import { SessionManager } from "@mariozechner/pi-coding-agent";
```

他停下来.

"**外部 npm 包**."

他 没 立刻 反应过来 这 件 事 的 含义. 他 先 在 OpenClaw 的 `src/` 里 grep 了 一下:

```bash
$ grep -rn "while\|for(;;)" src/agents/run/ | head
```

——**没有 一 行 真正 的 主 loop**. 全是 retry, 全是 try/catch, 全是 fallback. 没有 那个 "model.generate → tools.call → loop back" 的 循环 体.

他 又 在 整个 src/ 里 grep "stop_reason"——很多 reads, **零 writes**. 状态机 在 别的 地方 推进 的.

他 才 真正 反应 过来——

> ……**等等. loop 不在 我家**?

他打开 `package.json`——

```json
"@mariozechner/pi-coding-agent": "0.66.1",
```

"原来如此."

OpenClaw **不自己写 agent loop**. 它 import 了 `pi-coding-agent` 作为 loop 本体. **OpenClaw 自己那 5000+ 行, 全是 loop 的外围**.

他 盯 着 屏幕 看 了 大概 十 秒钟, 没 动.

——他 上 一家 公司 的 那个 chatbot, 他 自己 手写 过 loop. 三十 行. 他 一直 以为 OpenClaw 这 5000 行 是 那 三十 行 的 "工业级 加固版"——更 严谨 的 状态机, 更 多 的 边界情况.

不 是 这样. **OpenClaw 完全 没 写 那 三十 行**. 那 三十 行 它 委托 给 别人 了.

他 站 起来.

---

## 二

他走到厨房, 给自己倒了一杯水.

"我一直以为 OpenClaw 最核心的东西是它的 agent loop." 他自言自语. "我错了."

他靠着冰箱, 慢慢想.

**OpenClaw 的核心价值不是 loop, 是 loop 外围的所有东西.**

- 5 种 retry 策略
- Model fallback 链
- Auth profile rotation (多账号 + cooldown)
- Compaction (context 超限压缩)
- Delivery plan resolution
- Events bus (可观测性)
- Channel 分发 (23 个)
- Plugin SDK 版本化契约

**这些才是 OpenClaw 独有的**. Loop 本身, `pi-coding-agent` 也能提供.

---

## 三

他回书房, 打开新的一页笔记本, 画了一张图——

```
╔══════════ L3 · CLI 编排层 (1111 行) ══════════╗
║  agent-command.ts                              ║
║  - session/skills/model/auth/delivery          ║
║  - runWithModelFallback                        ║
╚══════════════════╤═════════════════════════════╝
                   ↓
╔══════════ L2 · Attempt 分发层 (~5000 行) ════╗
║  runAgentAttempt (CLI vs Embedded 分支)        ║
║  runEmbeddedPiAgent (外层 retry loop)         ║
║  runEmbeddedAttempt (单次 turn 的 2486 行编排)║
╚══════════════════╤═════════════════════════════╝
                   ↓
╔══════════ L1 · Loop 本体 (npm 外部) ═════════╗
║  @mariozechner/pi-coding-agent                 ║
║  SessionManager · stop_reason 状态机           ║
║  真正的 while(...) 在这里                     ║
╚════════════════════════════════════════════════╝
```

三层. 每层负责不同的事.

- **L3** 是给人用的界面, 是 orchestration.
- **L2** 是给 L3 准备"可重试的 agent run"抽象, 是 robustness.
- **L1** 是真正让 LLM 转起来的那个循环, 是 raw loop.

---

## 四

他突然理解为什么这个架构比他想象的合理——

**Loop 是被反复研究过的玩意. 写 loop 不是 OpenClaw 的核心能力, 别人写得更好也更专注.**

**外围才是 OpenClaw 的护城河.** Retry 策略、auth rotation、channel 分发、plugin SDK——这些在每个产品公司都要花几个月开发的东西, OpenClaw 把它们工程化了.

> **"用外部包做 loop, 自己专注 orchestration"——这是 2026 年 production 软件最常见的分层方式.**

像一个餐厅不自己种菜, 专心做菜. 菜更专业, 厨师更专业, 各自精进.

---

## 五

他在笔记页底部写了一段长话:

> 我之前以为 agent 技术的价值在 loop. 实际上 loop 是 2000 年代的 event loop 的 LLM 版, 已经**没有秘密**.
>
> Agent 技术的价值在于: **当这个 loop 在真实世界里跑 10000 次时, 它不掉链子**.
>
> 不掉链子靠 5 件事:
> 1. 失败了怎么重试 (5 种 retry)
> 2. 一个 provider 挂了怎么办 (model fallback)
> 3. 多账号怎么轮转 (auth profile rotation)
> 4. Context 超了怎么压缩 (compaction)
> 5. 回复怎么保证到达 (delivery)
>
> 这 5 件都在 L2/L3. **L1 只要"能跑就行"**.
>
> **这是一个反直觉的价值分布**——
>
> 越靠近用户能看见的地方, 价值反而越薄;
> 越往上游的可靠性工程走, 价值越厚.

他合上笔记本. 凌晨五点.

bug 还没修. 但他已经知道**今晚 75% 的失败率藏在 L2 的哪一层**. 凌晨的剩余时间, 他需要追那 5 件事里具体哪一件断了.

---

## 关键概念

- **三层分层**: L3 编排 / L2 attempt 分发 / L1 loop 本体.
- **Loop 外置**: `@mariozechner/pi-coding-agent` 提供 SessionManager + stop_reason 循环.
- **反直觉价值分布**: 可见的 loop 薄, 不可见的可靠性厚.

## 我的理解

- **Agent = Loop + Orchestration**. Loop 是公共品, Orchestration 是护城河.
- 想评估一个 agent 产品, 不看 loop 写得多漂亮, 看**它在生产环境里怎么处理失败**.
- 外置 loop 是**工程成熟**的标志——明白什么自己做、什么委托别人.

## 类比

像一个厨师——

- 菜市场(pi-coding-agent) 卖最好的肉
- 厨师(OpenClaw) 不自己养牛
- 厨师 90% 的价值在**怎么炒出上百桌, 每一桌都稳定出品**——这是火候管理, 不是肉源

你让厨师自己养牛, 他就没精力做菜了.

## 类比 · ASCII 版

```
    ┌─────────────────────────┐
    │  用户看见的 "Agent 表现" │  ← 这里的价值是 "稳定" (OpenClaw 贡献)
    └────────────┬────────────┘
                 │
       ┌─────────┴─────────┐
       │  Raw agent loop   │  ← 这里的价值是 "能跑" (pi-coding-agent 贡献)
       └───────────────────┘
```

## 遗留问题

- 75% 失败具体在哪一层? L2 的 5 种 retry 里有没有**失败后 lifecycle.end 不被 emit** 的 path? 这是我下一步要查的.
- 哪一天需要自己写 L1 loop? 如果 pi-coding-agent 的某个 bug 我修不了, 或者它不支持某个我们要的 feature——那时候才自己写.

## 与其他章节的连接

- ← [[ch01-brain|第一幕]]
- → [[ch03-manifest|第三幕]]: 另一个让我震惊的发现——**agent 看到的 tool 列表, 根本不在代码里决定**. 是 manifest 说的.
