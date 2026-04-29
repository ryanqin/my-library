# 编排国 · stop_reason 的王宫

> 这章要解决什么问题？
> Agent 编排是什么？`stop_reason` 为什么是那座王宫的王座？为什么"用 prompt 强制一件事"是反模式？

---

## 一

旅行者走进编排国首都——stop_reason 城——的那一刻，他迷了一下方向。

城里每一栋建筑都在**跳动**。

一栋楼顶写着 `tool_use`，它正在亮，里面的工人搬运着工具；下一秒它熄了，旁边一栋 `end_turn` 的楼亮起来，工人开始收摊；再下一秒 `max_tokens` 亮了一下——楼里的工人一脸困惑地看着半成品的货——"没做完啊？"

"这座城的节奏，" 向导对他说，"**就是 stop_reason**。"

旅行者掏出笔记本，写下他进城后的第一个定义：

> **Agentic loop = `while stop_reason != end_turn: 处理上一个 reason，再 ask claude`**

一个 if/else 链的状态机。**比他想象的朴素 10 倍**。

---

## 二

他在王宫广场上看到一面反模式墙。

墙上一行巨大的字:

> **"用 prompt 强制一件事"**

下面是失败案例陈列——

- 有人在 system prompt 里写 "**务必**在回复前调用 `check_policy` 工具"。结果 LLM 有时候记得，有时候忘——**成功率 85%**。
- 有人在 user turn 加一行"如果用户提到钱，必须先调用 `log_audit`"。结果 LLM 在 70% 的场景里记得，在 30% 的场景里被 prompt 冲淡了。

旁边有一块小牌子——"**正确姿势**"：

> **用 Hook，不用 Prompt**。

Hook 是代码层强制。Hook 里写：

```
on_tool_use(tool_name):
    if tool_name == "make_payment" and not has_called("check_policy"):
        reject with "Policy not checked"
```

这是**确定性**。Prompt 是**概率**。

旅行者在反模式墙前抄下这条铁律:

> **关键业务规则用 Hook 强制, 不要靠 prompt 请求.**
> Prompt 是建议, Hook 是法律.

---

## 三

他在王宫里第二个学到的是 **Coordinator-Subagent 模式**。

王宫大厅里有一个 coordinator agent——一个戴着金冠的老者。他周围有 5 个 subagent，分别去做 5 件事。

"为什么不让 coordinator 自己做？" 旅行者问。

老者笑了。"因为**每个 subagent 有自己的上下文**。"

他取出一张纸，画给旅行者看：

```
Coordinator
  ├── Subagent A (看 200 页合同)
  ├── Subagent B (查 100 条交易记录)
  ├── Subagent C (调 external API 取 50MB 报表)
  ├── Subagent D (走 plan 检查)
  └── Subagent E (汇总前 4 个, 产出 1 页结论给用户)
```

"如果让 coordinator 自己做，" 老者说，"它会吃下 200 + 100 + 50MB = 爆炸的 context。它会**在中间丢信息**。"

"Hub-and-spoke 的核心不是分工，" 他说，"**是上下文隔离**。"

旅行者懂了——subagent 不是"请来的帮手"，是**一个新的干净context 容器**，用来挡住"主上下文被污染"这件事。

---

## 四

编排国最后一个教学点是——**任务分解**。

王宫墙上挂着两张画：

- 左图: **Prompt chaining** — 一个任务拆成 3 个预定义步骤, 按顺序跑
- 右图: **动态分解** — agent 运行时自己决定要不要再拆

"什么时候用哪个?" 旅行者问.

"确定性高, 步骤少, 用 prompt chaining. 确定性低, 步骤数量不定, 用动态分解." 向导说.

"简单说: **已知的复杂度用 chain, 未知的复杂度用 decompose**."

---

## 关键概念

- **`stop_reason` 状态机**：`end_turn` / `tool_use` / `max_tokens` / `refusal` — 整个 agent loop 被这四个值驱动。
- **Hooks vs Prompt 强制**：关键业务规则用代码 Hook（确定性），不要靠 prompt（概率性）。
- **Coordinator-Subagent 模式**：Hub-and-spoke，核心价值是**上下文隔离**不是分工。
- **任务分解**：Chain（已知复杂度）vs Dynamic decomposition（未知复杂度）。

## 我的理解

编排国的四节课，反过来读是:
1. Agent 是**状态机**, 不是会话.
2. 强制用代码, 不要用嘴.
3. Subagent 是 context 容器, 不是打工仔.
4. 拆任务时问自己——"复杂度是已知还是未知?"

## 类比

stop_reason 城像一个**交通信号塔**——每个路口的红绿黄灯告诉你现在要做什么。Hook 是**路上的物理护栏**——不管你司机是不是看信号,车就是过不去. Prompt 是**路边的建议牌**——多数人会照做,少数人会无视.

Coordinator-Subagent 是**市长办公室 vs 各局**的关系——市长不自己审所有文件,而是让各局看完给个摘要上来.

## 遗留问题

- `stop_reason` 的第五种情况——`refusal`——在实际 agent 里怎么处理？是重试还是立刻停？
- Plan Mode 算不算一种 coordinator-subagent 模式的变形？我觉得算——plan 阶段是 coordinator，execute 阶段是 subagent。

## 与其他章节的连接

- ← [[ch00-map|序章 · 地图]]
- → [[ch02-tools|工具国]]：编排是让 agent 循环，但每一轮里它要调用的工具长什么样？
